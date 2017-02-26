---
title: Creating A Drag-n-Drop Enabled Text Box in WPF
tags: WPF, C#, .NET
---

Recently, a WPF desktop application that I was working on required the user 
to select multiple files, which were typically in different directories. I had 
implemented a "Browse" button that would allow the user to select these files 
via a file selection dialog box . However, this becomes very tedious when you 
are constantly selecting these different files. I decided to implement drag-n-drop
to make my users' lives easier.

There are a few different approaches that you can take. You can use 
a framework, such as the wonderful "GongSolutions.WPF.DragDrop" 
library. This library is great if you require more advanced
drag-n-drop functionality. I was originally using this library to implement 
the functionality that I needed, but later realized it was overkill. I 
decided to go with a different approach. I decided to create a custom control
that derived from the ```TextBox``` control.

First, you will need to override the ``OnApplyTemplate`` method of 
the ```TextBox``` control.  ```OnApplyTemplate``` is called just before the
UI element is displayed in the application. This is a good place to
wire the drag/drop related events:  
  
```cs
public override void OnApplyTemplate()
{
    // Make sure to call the base.OnApplyTemplate() first!
    base.OnApplyTemplate();

    DragEnter += FilePathTextBox_DragEnter;
    Drop += FilePathTextBox_Drop;
    PreviewDragOver += FilePathTextBox_PreviewDragOver;
}
```
  
We will now start implementing the event handlers that we specified 
in the ```OnApplyTemplate``` method. Let's start with the easy one 
first: ```FilePathTextBox_PreviewDragOver```. This handler tells the 
control to show the user the little '+' sign when he / she drags a file 
onto the text box:

```cs
private void FilePathTextBox_PreviewDragOver(object sender, DragEventArgs e)
{
    e.Handled = true;
}
```

Now, let's write the event handler that handles the event when the user drags 
a file on to the custom text box:  
  
```cs
private void FilePathTextBox_DragEnter(object sender, DragEventArgs e)
{
    var dragFileList = ((DataObject)e.Data).GetFileDropList().Cast<string>().ToList();
    var draggingOnlyOneFile = dragFileList.Count == 1 && dragFileList.All(item =>
    {
        var attributes = File.GetAttributes(item);
        return (attributes & FileAttributes.Directory) != FileAttributes.Directory;
    });

    e.Effects = draggingOnlyOneFile ? DragDropEffects.Copy : DragDropEffects.None;
}
```

The ```FilePathTextBox_DragEnter``` event handler gets a list of files that the user 
dragged on to the text box. In the code above, we want the user to only be able to 
drag 1 file (not directory) on to the text box. This filtering can be tailored to your 
needs. This is left as an exercise to the reader.  
  
Here is the last event that we need to implement:  

```cs
private void FilePathTextBox_Drop(object sender, DragEventArgs e)
{
    var dragFileList = ((DataObject)e.Data).GetFileDropList().Cast<string>().ToList();
    var draggingOnlyOneFile = dragFileList.Count == 1 && dragFileList.All(item =>
    {
        var attributes = File.GetAttributes(item);
        return (attributes & FileAttributes.Directory) != FileAttributes.Directory;
    });

    e.Effects = draggingOnlyOneFile ? DragDropEffects.Copy : DragDropEffects.None;

    // Set the Text property of the custom text box to the path of
    // the file the user dropped.
    if (draggingOnlyOneFile)
        Text = dragFileList[0];
}
```

Here is the final, refactored implementation:  
  
```cs
public class FilePathTextBox : TextBox
{
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        DragEnter += FilePathTextBox_DragEnter;
        Drop += FilePathTextBox_Drop;
        PreviewDragOver += FilePathTextBox_PreviewDragOver;
    }

    private void FilePathTextBox_PreviewDragOver(object sender, DragEventArgs e)
    {
        e.Handled = true;
    }

    private void FilePathTextBox_DragEnter(object sender, DragEventArgs e)
    {
        var didUserOnlyDragOneFile = DidUserDragOnlyOneFile(e);
        SetDragDropEffect(didUserOnlyDragOneFile, e);
    }

    private void FilePathTextBox_Drop(object sender, DragEventArgs e)
    {
        var didUserOnlyDragOneFile = DidUserDragOnlyOneFile(e);
        SetDragDropEffect(didUserOnlyDragOneFile, e);

        if (draggingOnlyOneFile)
            Text = dragFileList[0];
    }

    private bool DidUserDragOnlyOneFile(DragEventArgs e)
    {
        var dragFileList = ((DataObject)e.Data).GetFileDropList().Cast<string>().ToList();
        var draggingOnlyOneFile = dragFileList.Count == 1 && dragFileList.All(item =>
        {
            var attributes = File.GetAttributes(item);
            return (attributes & FileAttributes.Directory) != FileAttributes.Directory;
        });

        return draggingOnlyOneFile;
    }

    private void SetDragDropEffect(bool shouldShowCopyEffect, DragEventArgs e)
    {
        e.Effects = shouldShowCopyEffect ? DragDropEffects.Copy : DragDropEffects.None;
    }
}
```

Cheers,  

Connor Moreside 
