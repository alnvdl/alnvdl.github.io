---
layout: post
title: "Designing furniture using the CSS grid"
date: "2023-01-07 14:00:00 -0300"
excerpt: The story of why and how I developed a tool for designing MDF furniture by writing my own little language that transpiles into a CSS grid layout.
---

![A screenshot of the squareplanner web-based furniture design tool](/assets/img/squareplanner_screenshot.png)
*Live example: [https://alnvdl.github.io/squareplanner/](https://alnvdl.github.io/squareplanner/)*

At one point during the construction of my house, we needed designs for the kitchen and bathroom cabinets and a bedroom wardrobe.

In my country, this type of furniture is usually mounted under countertops or onto the bedroom walls using [MDF](https://en.wikipedia.org/wiki/Medium-density_fibreboard) boards. This type of material lends itself to a lot of customization.

People usually pay specialized professionals (or companies) to design and build this kind of MDF furniture for their houses, and it can go from very cheap to very expensive depending on one's choices of materials and finishing. These designers use specialized software (usually a 3D CAD), and the output is not only the design, but a detailed bill of materials. This output then gets sent to a factory, which cuts, finishes and delivers the boards and related materials directly to the destination, where it's usually assembled by people working for the company that designed the furniture.

As we did with the house blueprint, we wanted full control over every part of the design. We found it's much easier to make the designs ourselves, rather than imposing vetoes and "suggestions" on a designer's work, which can be annoying for both the designer and for ourselves.

I wasn't going to learn a bespoke 3D CAD software just for this. But I still needed a way to clearly communicate my ideas to the person designing and overseeing the manufacturing of our furniture. 2D was enough, as the depth of the furniture was fixed, dependent on where it is being placed.

My wife started drawing her ideas in Adobe Illustrator, but it was hard work. So you just noticed you need to add a new drawer? Now you have to redraw and move the other pieces around. Did you just realize that your stand mixer won't fit in a module that is only 35cm tall? You may need to rethink the whole cabinet... My wife and I would discuss an idea, and then it'd take her some manual work to redraw the whole thing. Repeat that a few times, and we were spending a lot of time on this.

One important realization struck me when looking at her designs though: most of the time, it's basically rectangles fitted together in a delimited area. And when designing it, if we changed our minds, these rectangles should respond to changes according to certain rules. For example, I might want that drawer to be 38cm wide, but I don't care about the size of the doors next to it.

At first I thought of writing a Python script that would take a text description, calculate a few things and output measurements in text. But that would not be very visual, so it was hard to grasp for both ourselves and the furniture designers. Plus, I'd have to write some code that fits rectangles. I was losing motivation, as it would require some non-trivial work, and the end result would not look nice anyway. Was there any software I knew that did at least some of this work for me?

**What if I could be lazy and reuse an existing calculation engine, while at the same time getting a nice graphical output?** Of course, I later realized that instead of writing a calculator I ended up having to write a transpiler, but then I was too involved by then to give up.

It turns out that fitting rectangles if basically what modern web browers do. Web developers and designers have been fitting rectangles in complex ways for a while now with concepts such as the CSS [flexbox](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Flexible_Box_Layout/Basic_Concepts_of_Flexbox) and [grid](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Grid_Layout) layouts. Professionally, I'm mostly a backend developer, but I venture myself in web development from time to time, and I had read great things about the CSS grid layout, so I decided to give it a try.

After studying the [CSS grid layout on MDN](https://developer.mozilla.org/en-US/docs/Learn/CSS/CSS_layout/Grids), I realized it was the perfect fit. I could tell the browser:
> this column is 78px tall, and I want 4 rows that are 38px wide in it

Or in other words:
> this module is 78cm tall, and I want 4 drawers that are 38cm wide in it

Of course, I was not going to write HTML and CSS by hand, as that is not much better than dragging boxes in Illustrator. What about the input format for that stillborn Python script? At first, I came up with something like this:
```
cabinet 200cm x 95cm top 3cm bottom 14cm scale 5
    38cm
*   drawer
*   drawer
*   drawer
*   drawer
```

In other words: my cabinet should be 200cm wide and 95cm tall. There will be a countertop taking 3cm at the top, and a base mount taking 14 cm at the bottom. I want a module that is 38cm wide, and I want 4 drawers in it, and I don't care about their height as long at they fit. The word `drawer` doesn't have any special meaning, and any word can be used to identify the rectangles.

As I kept on adding things, I needed more expressiveness to do the things I wanted:
```
cabinet 200cm x 95cm top 3cm bottom 14cm scale 5
    38cm     *      *      *        38cm
*   drawer   door   door   drawer   drawer:red
*   drawer   ^      ^      drawer   <
*   drawer   ^      ^      drawer   <
*   drawer   ^      ^      drawer   <
```

In this example we can see a few new things:
- `*` means "I don't care, make it fit for me";
- A number followed by `cm` is a required width or height (in centimeters);
- `<`, `>` and `^` are saying "continue whatever is to the left/right/above";
- Adding `:red`, `:green` or `:blue` after the name of a rectangle makes it have that color;
- The `scale` is mostly a hacky way of hinting how big things should be so that labels fit into rectangles;

Of course, I didn't develop this little language beforehand, as I mostly didn't know what I was doing at first. I was basically discovering new things I needed to express when designing new parts, and I then extended the code until it did everything I wanted.

It turns out this little language was enough to describe everything I needed, including my ridiculously over-engineered wardrobe [external](https://alnvdl.github.io/squareplanner/#lock:QXJt4XJpbyBkZSBjYXNhbCAodmlz428gZXh0ZXJuYSkgY29tIDM0MGNtIHggMjYwY20gYmFzZSBkZSAxNSBlbSBlc2NhbGEgMwogICAgICAgNDVjbSAgICA0NWNtICAgICogICAgICAgICAgICAgICAgKiAgICAgICA0NWNtICAgIDQ1Y20gICAgNDVjbQo2MGNtICAgcG9ydGEgICBwb3J0YSAgIHBvcnRhICAgICAgICAgICAgcG9ydGEgICBwb3J0YSAgIHBvcnRhICAgcG9ydGEKKiAgICAgIHBvcnRhICAgcG9ydGEgICBuaWNob190djp2ZXJkZSAgIDwgICAgICAgcG9ydGEgICBwb3J0YSAgIHBvcnRhCjQwY20gICBeICAgICAgIF4gICAgICAgXiAgICAgICAgICAgICAgICA8ICAgICAgIF4gICAgICAgXiAgICAgICBeCjIwY20gICBeICAgICAgIF4gICAgICAgbmljaG9fYWJlcnRvICAgICA8ICAgICAgIF4gICAgICAgXiAgICAgICBeCjMwY20gICBeICAgICAgIF4gICAgICAgZ2F2ZXRhICAgICAgICAgICA8ICAgICAgIF4gICAgICAgXiAgICAgICBeCjE1Y20gICBeICAgICAgIF4gICAgICAgZ2F2ZXRhICAgICAgICAgICA8ICAgICAgIF4gICAgICAgXiAgICAgICBeCjE1Y20gICBeICAgICAgIF4gICAgICAgXiAgICAgICAgICAgICAgICA8ICAgICAgIF4gICAgICAgXiAgICAgICBeCjE1Y20gICBeICAgICAgIF4gICAgICAgZ2F2ZXRhICAgICAgICAgICA8ICAgICAgIF4gICAgICAgXiAgICAgICBeCjE1Y20gICBeICAgICAgIF4gICAgICAgZ2F2ZXRhICAgICAgICAgICA8ICAgICAgIF4gICAgICAgXiAgICAgICBeCgojIEVtIHZlcmRlLCBuaWNobyBkZSBUViBjb20gcHJvZnVuZGlkYWRlIGRlIDMwY20uIFbjbyBsaXZyZSBkZSBvdXRyb3MgMzBjbSBlc2NvbmRpZG8gYXRy4XMsIGUgbmljaG8gYWJlcnRvIGxvZ28gYWJhaXhvLgojIFByb2Z1bmRpZGFkZSBkbyBhcm3hcmlvOiA2MGNtLg==) and [internal](https://alnvdl.github.io/squareplanner/#lock:QXJt4XJpbyBkZSBjYXNhbCAodmlz428gaW50ZXJuYSkgY29tIDM0MGNtIHggMjYwY20gYmFzZSBkZSAxNSBlbSBlc2NhbGEgMwogICAgICAgNDVjbSAgICAgNDVjbSAgICAgICAgICogICAgICAgICAgICAgICAgICAgICAgICAgICAgIDQ1Y20gICA0NWNtICAgICAgICAgNDVjbSAgIDQ1Y20KNjBjbSAgIG5pY2hvICAgIDwgICAgICAgICAgICBuaWNobyAgICAgICAgICAgICAgICAgICAgICAgICA8ICAgICAgbmljaG8gICAgICAgIDwgICAgICBuaWNobwoqICAgICAgbmljaG8gICAgPCAgICAgICAgICAgIG5pY2hvX3R2K25pY2hvX2FiZXJ0bzp2ZXJkZSAgIDwgICAgICBuaWNobyAgICAgICAgPCAgICAgIHBlbmR1cmFkb3IKNDBjbSAgIG5pY2hvICAgIHBlbmR1cmFkb3IgICBeICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8ICAgICAgcGVuZHVyYWRvciAgIDwgICAgICBeCjMwY20gICBeICAgICAgICBeICAgICAgICAgICAgXiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgXiAgICAgIF4gICAgICAgICAgICBeICAgICAgXgozMGNtICAgbmljaG8gICAgXiAgICAgICAgICAgIGdhdmV0YSAgICAgICAgICAgICAgICAgICAgICAgIDwgICAgICBeICAgICAgICAgICAgPCAgICAgIF4KMTVjbSAgIGdhdmV0YSAgIDwgICAgICAgICAgICBnYXZldGEgICAgICAgICAgICAgICAgICAgICAgICA8ICAgICAgZ2F2ZXRhICAgICAgIDwgICAgICBeCjE1Y20gICBnYXZldGEgICA8ICAgICAgICAgICAgXiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPCAgICAgIGdhdmV0YSAgICAgICA8ICAgICAgXgoxNWNtICAgZ2F2ZXRhICAgPCAgICAgICAgICAgIGdhdmV0YSAgICAgICAgICAgICAgICAgICAgICAgIDwgICAgICBnYXZldGEgICAgICAgPCAgICAgIF4KMTVjbSAgIGdhdmV0YSAgIDwgICAgICAgICAgICBnYXZldGEgICAgICAgICAgICAgICAgICAgICAgICA8ICAgICAgZ2F2ZXRhICAgICAgIDwgICAgICBe) designs.

Originally, input was hardcoded in the file. Then I moved it into an editable [`textarea`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/textarea), so that I could easily tweak it in the browser and see the results in realtime.

I also needed a way to easily share this with the person designing the furniture in the 3D CAD. I definitely didn't want to host this anywhere (I'm cheap). But I needed an easy and safe way to share this relatively complex data. There's a trick I have used in several projects before: store information in the [URL hash](https://developer.mozilla.org/en-US/docs/Web/API/URL/hash). JavaScript code can both write the hash it and read it. There's a [limitation of 2048 characters in URLs](https://stackoverflow.com/questions/417142/what-is-the-maximum-length-of-a-url-in-different-browsers), but if I couldn't describe my furniture in 2048 characters, I probably wouldn't be able to afford it anyway :)

When sharing a design, I didn't want people opening the page and accidentally pressing keys in the editor, breaking my carefully laid out rectangles and then delivering me a cabinet with giant drawers. If you press `Ctrl`+`Alt`+`L` on the page, it will lock the text area to prevent editing, and persist that lock state into the URL so it can be safely shared.

Ah, and I also realized it would be useful to have comments for myself and for the people reading the designs. So I added comments to the mini language (lines beginning with `#`). And colors too, as I needed to highlight a few things in the diagram so people could relate them to the comments (for example, `drawed:red` will make a red drawer).

And I'd need to print and export this to PDF, to carry with me, make notes and adaptations as I decided on the designs. So there's a bit of CSS for adapting the page for nicer print output, and the browser then takes care of printing to PDF.

Finally, it was getting really annoying coding in that language, full of whitespaces and columns and rows. By this point, it had become a pet project, so why not just add code formatting? Pressing `Ctrl`+`Alt`+`F` in the editable `textarea` will cause the code to be formatted in columns.

And that's it. I spent around 5 evenings working on this. I think I would actually have spent more time dragging rectangles with my wife in Illustrator, given how slow we were (and how much we were arguing over every decision). At least with this tool, it became extremely cheap to prototype weird cabinet designs together, only to realize they were actually not that bad.

You can play with this tool yourself: [https://alnvdl.github.io/squareplanner/](https://alnvdl.github.io/squareplanner/)

There is no server-side code, just static HTML, JavaScript and CSS, and it is hosted as a static page.

It's not written using any frameworks, if you push it too hard it will draw weird things, it's not formally tested, and it does not follow a particularly academic/professional approach to the transpilation process. I guess making it more "professional" would have been perfectly fine, but in this case I just wanted to get my furniture designed fast, and keeping the code simple and small helped in that. At the time I designed this, I had dozens of other problems to solve in the construction. Looking at the end result, it's amazing to to see all the complexity supported by modern browsers, which enable an application like this to be written in just ~550 non-minified LOC (or ~18KB).

In the end, the person designing the furniture in the 3D CAD had very few questions and was able to do their work. This design was also used by the company making the countertops, and we also used it throughout the construction process to help define where to place power outlets and pipes behind the cabinets (yes, we took the whole "design the house of your dreams" thing a bit too seriously).

And the end result was exactly as we wanted. I'm sharing this in the hope that it can be useful for someone else in the same situation as I was, or maybe help someone who is learning about the CSS grid layout.
