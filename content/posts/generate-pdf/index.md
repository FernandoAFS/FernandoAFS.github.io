+++
title = "Christmas card factory: How to generate pdfs in Python"
description = "Let's compare how to generate documents and find the most convenient way"
date = 2023-12-30T12:14:50+01:00
type = "post"
draft = false
image = "etsy_christmas_card.jpg"
tags = ["pdf", "templating", "python", "libreoffice"]
+++

[TL;DR: Check out the code.](https://github.com/FernandoAFS/pdf-christmas-card-factory)

# Introduction


I'm making a christmas card pdf factory. There are many ways to go about it. Let's find out how.

![Let's do code as craft. source: Etsy](etsy_christmas_card.jpg)
*Credit to EdenwoodPaperie. Published on [Etsy](https://www.etsy.com/listing/889493692/folded-christmas-card-template-5x7-happy)*

# State of the art

I was making a Python application so this is what I found:

I haven't even tried the following:

- [myPDF](https://pypi.org/project/pypdf/): It's about manipulating pdf files, not generating new ones.
- [Pylatex](https://pypi.org/project/PyLaTeX/): Good for papers, not the best for a Christmas card!
- [ReportLab](https://pypi.org/project/reportlab/): To be hones I found it way to complex and it's focused on reporting.

## xhtml2pdf

> xhtml2pdf enables users to generate PDF documents from HTML content easily and with automated flow control such as pagination and keeping text together.

[Xhtml2pdf](https://pypi.org/project/xhtml2pdf/) is a very interesting tool that creates PDF from html technologies.

I'm not good with CSS and this is not an easy one. There is a [good post on MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_media_queries/Printing) if you are interested.

There is nothing wrong with this one but I've spent a whole day trying to produce something good looking without luck.

I would recommend to check this out if:

1. You are somehow good writing raw CSS.
2. You come from a HTML template.

## Weasyprint

> WeasyPrint is a smart solution helping web developers to create PDF documents. It turns simple HTML pages into gorgeous statistical reports, invoices, tickets…

[Weasyprint](https://pypi.org/project/weasyprint/) is very similar to the last one with prettier documentation and more "modern" feel. But I has the problems as the first one.

# Looking for another way

Maybe It's better not to do too much pdf on Python...

I found a [post at "python is spoken here"](http://blog.pyspoken.com/2016/07/27/creating-pdf-documents-using-libreoffice-and-python/) about how to create a PDF. This article is great, however It has been written in 2016, It didn't work right and I think It can be made easier.

This is what I would recommend:

1. Export the document to .fodX format instead of the regular .odX. This creates a flat and self-contained file.
2. Create an XML template from that.
3. Use [Unoserver](https://github.com/unoconv/unoserver/) and xmlrpc to generate the PDF.

# Let's start coding

I'm going to create a personalized Christmas card.

You can find everything in my repo.

## Create the template document

For this I'm going to use a [template from slidesgo](https://slidesgo.com/theme/cute-christmas-cards-collection).

**Note**: If you come from a PowerPoint slide I'd recommend to
create a new new libreoffice document and to copy-and-paste manual each slide you are going to use one by one and to re-insert every image that is going to be changed programmatically. Libreoffice generates much simpler XML templates when used natively.

![Merry Christmas Tim Cook](tim-cook-christmas.jpg)
*Whish you a great christmas mr Tom Cook*

Let's [save this](document.fodp) as .fodp (Flat XML ODF Presentation).

## From XML to template.

Let's open this file and change some stuff...

In the line 6218 i find:

```XML
<text:p text:style-name="P16"><text:span text:style-name="T2">
    Hope you are having a great time Tim Cook
</text:span></text:p>
```

In the line 10336 start the file object. If we copy the base64 string and use the following command we can see that it's just a png image:

```bash
< b64_contents.txt base64 --decode | file -
```

In this example I'm using Jinja2 but any template language would be fine.

The python scripts looks like:

```python
# christmas-card-factory.py
#!/bin/env python3

import argparse
import jinja2
import base64
import textwrap
import sys

# THERE IS ONLY ONE TEMPLATE FILE IN THIS EXAMPLE.
env = jinja2.Environment(
    loader=jinja2.FileSystemLoader("./")
)

def photo_b64(src):
    with open(src, "rb") as f:
        photo_bytes = f.read()

    photo_base64 = base64.b64encode(photo_bytes).decode()
    return textwrap.fill(photo_base64, 73)


def main():
    # ARGUEMNTS
    parser = argparse.ArgumentParser(
        prog="Christmas card generator",
        description="Use this to turn the fodp template to a pdf file!",
    )
    parser.add_argument("name", help="Name to be used in the template")
    parser.add_argument("photo", help="Path of the photo to be used as a portrait")
    parser.add_argument("--action", help="Filetype to be outputed (fodp or pdf), default pdf", default="pdf")

    args = parser.parse_args()

    name = args.name
    photo_path = args.photo
    action = args.action
    # END OF ARGUMENTS. ACUTAL CODE

    # template.xml is the same as the document linked above with the two arguments.
    # check the github repository for more details.
    template = env.get_template("template.xml")
    photo_base64 = b64image=photo_b64(photo_path)
    fodp_document = template.render(name=name, b64image=photo_base64)
    print(fodp_document)


if __name__ == "__main__":
    main()
```

This outputs an fodp file to stdout.

Now run the unoserver in another terminal. I'm using Arch and it's included with the libreoffice-fresh package.

```bash
unoserver
```

Let's check the pdf conversion. I'm reading from stdin directly but you can dump the output to a file and read it with your pdf reader of choice.

```bash
python christmas-card-factory.py --action fodp "Tim Cook" assets/Tim-Cook-Circle.png | unoconvert --convert-to pdf - - > tim_cook_merry_christmas.pdf
```

Now, let's see how to create documents programmatically.

## Automating the conversion to pdf

Thankfully unoserver is written in python. Let's use unoconvert to generate a pdf directly:

```python
#!/bin/env python3

import argparse
import jinja2
import base64
import textwrap
import sys
import unoserver.client

# THERE IS ONLY ONE TEMPLATE FILE IN THIS EXAMPLE.
env = jinja2.Environment(
    loader=jinja2.FileSystemLoader("./")
)

def photo_b64(src):
    with open(src, "rb") as f:
        photo_bytes = f.read()

    photo_base64 = base64.b64encode(photo_bytes).decode()
    return textwrap.fill(photo_base64, 73)


def main():
    parser = argparse.ArgumentParser(
        prog="Christmas card generator",
        description="Use this to turn the fodp template to a pdf file!",
    )
    parser.add_argument("name", help="Name to be used in the template")
    parser.add_argument("photo", help="Path of the photo to be used as a portrait")
    parser.add_argument("--action", help="Filetype to be outputed (fodp or pdf), default pdf", default="pdf")
    parser.add_argument("--unoserver-host", default="localhost")
    parser.add_argument("--unoserver-port", default="2003")

    args = parser.parse_args()

    name = args.name
    photo_path = args.photo
    action = args.action
    unoserver_host = args.unoserver_host
    unoserver_port = args.unoserver_port

    if action not in ["fodp", "pdf"]:
        raise ValueError(f"Action argument can either be fodp or pdf but not {action}")

    # template.xml is the same as the document linked above with the two arguments.
    # check the github repository for more details.
    template = env.get_template("template.xml")
    photo_base64 = b64image=photo_b64(photo_path)
    fodp_document = template.render(name=name, b64image=photo_base64)

    if action == "fodp":
        print(fodp_document)
        return

    client = unoserver.client.UnoClient(server=unoserver_host, port=unoserver_port)
    pdf_bin = client.convert(indata=fodp_document.encode(), convert_to="pdf")
    sys.stdout.buffer.write(pdf_bin)


if __name__ == "__main__":
    main()
```

This script is almost the same as the last one. But now it sends data directly to the server without going through unoconvert cli.

Now we can do:

```bash
python christmas-card-factory.py  --action pdf "Tim Cook" assets/Tim-Cook-Circle.png | zathura -
```

This is very powerful if you want to do this in an automated environment. To run this in a production environment just wrap unoserver in a micoservice and tinker a bit (if at all) with the client.

You can use an [executor](https://docs.python.org/3/library/concurrent.futures.html#threadpoolexecutor) to run the unoserver.client in an async environment or you can even write your own client with an async [xml-rpc library](https://pypi.org/project/async-rpc/).

# Wrapping up

You can check everything out [in my repo demo](https://github.com/FernandoAFS/pdf-christmas-card-factory).

I've used a presentation but you can apply the same principle for documents and spread sheets.

There is a few takeaways I got from all this:

- There is so much attention in web technologies that I forgot to look elsewhere.
- Sometimes the easy way turns very hard very quickly. There are pdf-generation libraries but I think this solution is easier for most "standard" pdf generation solutions.

I think this solution follows the "do one thing and do it right". Let an office suite generate the template. Use a template language to generate the document and finally use a server application to convert it to PDF. This is what good software should aim to be. **It's great to find an established technology and use it in a new way.**

There is no complex library, just plain old XML. Unoserver isn't particularly fancy either. It's a wrapper over libreoffie with an xml-rpc layer. If you already know-use template environments I would consider this approach to generate slight variations against other alternatives such as [openpyxl](https://pypi.org/project/openpyxl/) or [json2pdf](https://docs.reportlab.com/json2pdf/).

Again, thanks to [python spoken here](http://blog.pyspoken.com/) and to the [thread about libreoffice internals](>https://ask.libreoffice.org/t/how-can-i-uncompress-a-libreoffice-document-to-get-its-xml-internals-then-make-a-new-document-from-them/59721) that gave me the ideas in this blog.

Hope you found this helpful!
