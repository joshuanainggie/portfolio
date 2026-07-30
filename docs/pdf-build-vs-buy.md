---
description: Why I evaluated three commercial PDF form-filling services for a logistics workflow and ended up writing a hundred lines of Python instead.
---

# I Priced Three PDF Vendors, Then Wrote 100 Lines of Python Instead

**Subtitle:** Filling a PDF form turns out not to be document manipulation. It's assigning values to a dictionary.

---

Our dispatchers fill in airline shipment forms. Not now and then, but constantly, as part of every handoff to a carrier. One of those forms has 53 fields on it, and most of them are client and route details that already exist in our system. Somebody was retyping them into Acrobat by hand, several times a shift, on an operation that runs around the clock.

Obvious fix: fill the PDF programmatically. So I went looking for something to do it.

## Why I didn't buy one

If you're on the Microsoft stack there are three serious options. Adobe PDF Services, Plumsail Documents, Encodian. All three are competent, all three have a Power Automate connector that actually works, and that last point matters a lot when you're the only person maintaining the automation.

I want to be fair to them, so here's what actually ruled them out. It wasn't the sticker price.

The first thing was where the document goes. All of these work by shipping your file out to a third-party service and handing it back filled. For a shipment form that means client names, addresses and routing details leaving our tenant every single time. That's not a disaster, and plenty of companies accept the trade deliberately. But it becomes a data-residency conversation with the owners, then a vendor review, then a dependency I get to re-explain every time somebody new asks where client data goes. None of that appears on a pricing page and all of it is real cost.

The second thing was the shape of the pricing rather than the amount. These products bill per transaction or per volume tier. We're a courier operation. Volume is the thing we're actively trying to grow. I'd rather not put a model that charges me more as the business does better underneath a workflow this central.

So before signing anything I spent some time working out whether the problem was actually hard.

## The bit I didn't know: the fields already have names

Open a fillable PDF in more or less any library, enumerate the form widgets, and you get back the internal field names whoever designed the form assigned. Not coordinates. Not guesses about where text should land. Named fields, `ShipperName` and `AWB_Number` and so on, sitting there waiting for a value.

Which reframes the whole thing. Filling a PDF form isn't document manipulation. It's writing values into a dictionary and saving the file.

The first thing I built wasn't even the filler. It was a script that dumped every field name in the form to a CSV, and that CSV became a field catalog I could map our own data against. Fifty-three rows, generated in seconds, and suddenly the contract between our system and that form was written down instead of living in somebody's head.

That catalog turned out to be worth more than the filling code. It's what has to be redone for every new carrier form, and it documents itself.

## The build

FastAPI wrapping PyMuPDF, in a container. That's the whole service, about a hundred lines, and most of it is error handling and logging.

The core is this small:

```python
import fitz  # PyMuPDF

def fill_form(template_path: str, values: dict[str, str], out_path: str) -> int:
    doc = fitz.open(template_path)
    filled = 0

    for page in doc:
        for widget in page.widgets():
            name = widget.field_name
            if name in values:
                widget.field_value = str(values[name])
                widget.update()
                filled += 1

    doc.bake()  # flatten: widgets become page content
    doc.save(out_path, deflate=True, garbage=3)
    doc.close()
    return filled
```

Two lines in there deserve more attention than the rest of the file.

`widget.update()` is not optional and it's very easy to miss. Assigning `field_value` changes the object in memory. Without the update call, that change never reaches the saved file, and what you get is a PDF that opens perfectly and is completely blank. No error, nothing in the logs. If your output is empty, this is almost certainly why.

Then there's flattening. A filled form whose fields are still editable widgets isn't a finished document — the carrier receiving it can change your values, and PDF readers render form fields inconsistently enough that you can't be sure what they'll even see. `doc.bake()` turns the widgets into ordinary page content so what you send is fixed. That's the step that takes you from "a form with data in it" to "a document." It needs PyMuPDF 1.24 or newer.

## The result

Tested against the real form: 29 fields populated, flattened, no errors. Not a synthetic test file, the actual document we send to a real carrier, checked field by field against what a person would have typed.

Sub-second per document. No per-transaction cost. Client data never leaves our infrastructure. And when the carrier revises the form, I re-run the catalog extraction and diff the field names rather than filing a support ticket.

## Where I'd argue with myself

I kept one of the vendors as a documented fallback, and I'd suggest anyone doing this does the same. If PyMuPDF ever chokes on a form some carrier built in an unusual way, I want a second path that doesn't depend on me being awake.

More importantly, all of this only works because the forms are properly fillable. Get a flat scan with no form fields and none of the above applies; you're into OCR and coordinate mapping, which is a different project with a different budget. Check your actual documents have named fields before you assume you can skip the vendor.

And there's an honest counterargument to the whole post. A hundred lines of Python isn't zero maintenance. It's a container to deploy, patch and monitor, and it's mine forever. If you don't already run cloud infrastructure, or nobody on the team can support a Python service, the vendor connector is the right answer and the money is well spent. I happened to have the infrastructure and the ability to maintain it, and that changes the arithmetic completely.

## The general point

Before you evaluate vendors for a well-defined problem, spend a little time finding out how hard the problem actually is. Not to be clever about it, and not because building is always better. Just so you know what you're paying for.

Here the answer was that the PDF specification solved the hard part decades ago, and every vendor in this space is charging for a convenience layer over `field_value = x`. That convenience is worth real money to a lot of teams. It wasn't worth it to mine.

---

*I build and maintain the automation for a 24/7 medical courier operation: Microsoft 365, Azure, Power Platform, and whatever needs writing from scratch.*
