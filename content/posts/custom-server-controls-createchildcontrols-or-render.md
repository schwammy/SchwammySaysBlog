---
title: "Custom Server Controls: CreateChildControls() or Render()?"
slug: "custom-server-controls-createchildcontrols-or-render"
date: "2007-09-28T15:34:04\u002B00:00"
author: "schwammy"
categories: ["asp.net"]
---

This will be the first in series of posts I’ll be writing about creating server controls. I’ll try to share some of what I learned from my research while I prepared for a recent presentation.

With most software development, there isn’t always a simple answer to a question. This is of course true for creating Composite Controls for ASP.Net 2.0. One area where this was apparent was the proper way to render a control. Of the many examples I have found on the web it seems that *most* people put their rendering code within CreateChildControls() but some override the Render() method. I’ll show examples below. So which is correct? You know the answer…It Depends! My personal preference seems to be outside of the norm but I prefer to include rendering code in the Render() method (as in Option A below) unless there is a compelling reason not to.

My sample CompositeControl will have 2 controls within it:

```csharp
        private TextBox _textBox = new TextBox();
        private Label _label = new Label();
```

### Option A:

```csharp
        protected override void CreateChildControls()
        {
            Controls.Clear();
            _textBox.ID = "txtBox1";
            _label.ID = "label1";
            Controls.Add(_textBox);
            Controls.Add(_label);
            base.CreateChildControls();
        }
        protected override void Render(HtmlTextWriter writer)
        {
            EnsureChildControls();
            AddAttributesToRender(writer);
            writer.RenderBeginTag(HtmlTextWriterTag.Div);
            writer.AddAttribute(HtmlTextWriterAttribute.Cellpadding, "1", false);
            writer.RenderBeginTag(HtmlTextWriterTag.Table);
            writer.RenderBeginTag(HtmlTextWriterTag.Tr);
            writer.RenderBeginTag(HtmlTextWriterTag.Td);
            _label.RenderControl(writer);
            writer.RenderEndTag(); //</td>
            writer.RenderBeginTag(HtmlTextWriterTag.Td);
            _textBox.RenderControl(writer);
            writer.RenderEndTag(); //</td>
            writer.RenderEndTag(); //</tr>
            writer.RenderEndTag(); //</table>
            writer.RenderEndTag();
        }
```

In option A, CreateChildControls is used simple to set properties on the controls and add them to the Controls Collection. The Render() method takes care of layout. In particular, I like being able to use the HtmlTextWriterTag enumeration with the RenderBeginTag() method. By the way, the RenderEndTag() method will “close” whatever tag is next in line to be closed within the nested hierarchy of tags. Note that the Div tag I render isn’t really necessary. But I want all of my samples to render the same markup, so I put that in. Use the AddAttribute() method to add attributes to whichever tag is written next after the AddAtttribute() call. So in this sample, the CellPadding attribute will get written into the table as <table cellpadding=”1″>.

### Option B:

```csharp
        protected override void CreateChildControls()
        {
            Controls.Clear();
            _textBox.ID = "txtBox1";
            _label.ID = "label1";

            Controls.Add(new LiteralControl("<table cellpadding='1'><tr>"));
            Controls.Add(new LiteralControl("<td>"));
            Controls.Add(_label);
            Controls.Add(new LiteralControl("</td>"));
            Controls.Add(new LiteralControl("<td>"));
            Controls.Add(_textBox);
            Controls.Add(new LiteralControl("</td>"));
            Controls.Add(new LiteralControl("</tr></table>"));
            base.CreateChildControls();
        }
```

In option B, all of the rendering is done within CreateChildControls(). There is nothing wrong with this method, however, I don’t like it as much. I like my methods to do one thing only. And since there is a method named Render(), doesn’t it make sense to put the render code in *it*? Also, within CreateChildControls, you don’t get to use the RenderBeginTag() method with the enumerations. That means more chances for mistakes, in my opinion. If you are using this method within a CompositeControl, it is not necessary to override Render(). The default implementation of Render() will be called.

### Option C:

```csharp
        protected override void CreateChildControls()
        {
            Controls.Clear();
            _textBox.ID = "txtBox1";
            _label.ID = "label1";

            Table table = new Table();
            table.CellPadding = 1;
            TableRow row1 = new TableRow();
            TableCell cell1 = new TableCell();
            cell1.Controls.Add(_label);
            TableCell cell2 = new TableCell();
            cell2.Controls.Add(_textBox);
            row1.Cells.Add(cell1);
            row1.Cells.Add(cell2);
            table.Rows.Add(row1);

            Controls.Add(table);
            base.CreateChildControls();
        }
```

Lastly, I put Option C to show what NOT to do (most of the time). In this case, I have created my Table, Row and Cells as actual controls, not literals. This is extra overhead and it is not needed because the only purpose for my table is to control layout. The literal controls in options A and B will render quicker. However, there are times when you NEED to have your Table, etc, included in the Controls collection. In those cases, this technique can be used. Once again, there is no need to override the Render() method with this technique.

### Results:

So how does this all render in the browser? I put this markup into an ASP.Net Page:

```html
    <!--Option A -->
    <cc1:MyControl1 runat="server" ID="mc1" />
    <br />
    ------------------------------------------
    <!--Option B -->
    <cc1:MyControl2 runat="server" ID="mc2" />
    <br />
    ------------------------------------------
    <!--Option C -->
    <cc1:MyControl3 runat="server" ID="mc3" />
```

In the browser, I used “View Source” and got the following:

```text
        <div id="mc1">
            <table cellpadding="1">
                <tr>
                    <td>
                        <span id="mc1_label1"></span>
                    </td>
                    <td>
                        <input name="mc1$txtBox1" type="text" id="mc1_txtBox1" /></td>
                </tr>
            </table>
        </div>
        <br />
        ------------------------------------------
        <div id="mc2">
            <table cellpadding='1'>
                <tr>
                    <td>
                        <span id="mc2_label1"></span>
                    </td>
                    <td>
                        <input name="mc2$txtBox1" type="text" id="mc2_txtBox1" /></td>
                </tr>
            </table>
        </div>
        <br />
        ------------------------------------------
        <div id="mc3">
            <table cellpadding="1" border="0">
                <tr>
                    <td>
                        <span id="mc3_label1"></span>
                    </td>
                    <td>
                        <input name="mc3$txtBox1" type="text" id="mc3_txtBox1" /></td>
                </tr>
            </table>
        </div>
```

Looks pretty close, huh? There was one extra trick I did to make these match up. First, remember that in Option A, I added an extra <div> tag when I rendered my control. Without it, my option A control’s outer tag would have been <table>. Either way may work fine in your application. Second, Options B and C also have the following code included:

```csharp
        protected override HtmlTextWriterTag TagKey
        {
            get { return HtmlTextWriterTag.Div; }
        }
```

Without this override, my controls in Options B and C would have an outermost tag of <span> instead of <div>. Many people prefer to have controls render with <div> tags and this is how to do it!

I hope this information is helpful to you. Unfortunately, there isn’t always a clear answer of which method is best.
