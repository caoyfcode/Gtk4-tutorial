# Widgets (3)

## Open signal

### G\_APPLICATION\_HANDLES\_OPEN flag

GtkTextView, GtkTextBuffer and GtkScrolledWindow have given us a minimum editor in the previous section.
Next, we will add a function to read a file and remake the program into a file viewer.
There are many ways to implement the function.
Because this is a tutorial for beginners, we'll take the easiest one.

When the program starts, we give a filename as an argument.

    $ ./a.out filename

Then it opens the file and inserts its contents into the GtkTextBuffer.

At the beginning of the implementation, we need to know how GtkApplication (or GApplication) recognizes arguments.
It is described in the [GIO API Reference, Application](https://docs.gtk.org/gio/class.Application.html).

When GtkApplication is created, a flag (its type is GApplicationFlags) is given as an argument.

~~~C
GtkApplication *
gtk_application_new (const gchar *application_id, GApplicationFlags flags);
~~~

This tutorial explains only two flags, `G_APPLICATION_FLAGS_NONE` and `G_APPLICATION_HANDLES_OPEN`.
If you want to handle command line arguments, `G_APPLICATION_HANDLES_COMMAND_LINE` flag is what you need.
How to program it is written in [GIO API Reference, g\_application\_run](https://docs.gtk.org/gio/method.Application.run.html).
And the flag is described in the [GIO API Reference, ApplicationFlags](https://docs.gtk.org/gio/flags.ApplicationFlags.html).

~~~
GApplicationFlags' Members

G_APPLICATION_FLAGS_NONE  Default. (No argument allowed)
  ... ... ...
G_APPLICATION_HANDLES_OPEN  This application handles opening files (in the primary instance).
  ... ... ...
~~~

There are ten flags.
But we only need two of them so far.
We've already used `G_APPLICATION_FLAGS_NONE`.
It is the simplest option.
No argument is allowed.
If you give arguments and run the application, then error occurs.

`G_APPLICATION_HANDLES_OPEN` is the second simplest option.
It allows arguments but only files.
The application assumes all the arguments are filenames.

Now we use this flag when creating GtkApplication.

~~~C
app = gtk_application_new ("com.github.ToshioCP.tfv3", G_APPLICATION_HANDLES_OPEN);
~~~

### open signal

When the application starts, two signals are possible.

- activate signal --- This signal is emitted when there's no argument.
- open signal --- This signal is emitted when there is at least one argument.

The handler of "open" signal is defined as follows.

~~~C
void user_function (GApplication *application,
                   gpointer      files,
                   gint          n_files,
                   gchar        *hint,
                   gpointer      user_data)
~~~

The parameters are:

- `application` --- the application (usually GtkApplication)
- `files` --- an array of GFiles. [array length=n\_files] [element-type GFile]
- `n_files` --- the number of the elements of `files`
- `hint` --- a hint provided by the calling instance (usually it can be ignored)
- `user_data` --- user data set when the signal handler was connected.

The way how to read a file (GFile) will be described in the next subsection.

## Making a file viewer

### What is a file viewer?

A file viewer is a program that shows a text file given as an argument.
It works as follows.

- If it is given arguments, it recognizes the first argument as a filename and opens it.
- If opening the file succeeds, it reads the contents of the file and inserts it to GtkTextBuffer and shows the window.
- If it fails to open the file, shows an error message and quits.
- If there's no argument, it shows an error message and quits.
- If there are two or more arguments, the second one and after are ignored.

The program is as follows.

@@@include
tfv/tfv3.c
@@@

Save it as `tfv3.c`.
Then compile and run it.

    $ comp tfv3
    $ ./a.out tfv3.c

![File viewer](../image/screenshot_tfv3.png){width=6.3cm height=5.325cm}

Now I want to explain the program `tfv3.c`.
First, the function `main` changes in only two lines from the previous version.

- `G_APPLICATION_FLAGS_NONE` is replaced by `G_APPLICATION_HANDLES_OPEN`.
- `g_signal_connect (app, "open", G_CALLBACK (on_open), NULL)` is added.

Next, the handler `app_activate` is now very simple.
It just outputs the error message.
The application quits immediately because no window is created.

The point is the handler `app_open`.

- It creates GtkApplicationWindow, GtkScrolledWindow, GtkTextView and GtkTextBuffer and connects them.
- Sets wrap mode to `GTK_WRAP_WORD_CHAR` in GtktextView.
- Sets GtkTextView to non-editable because the program isn't an editor but only a viewer.
- Reads the file and inserts the text into GtkTextBuffer (this will be explained in detail later).
- If the file is not opened then outputs an error message and destroys the window. It makes the application quit.

The file reading part of the program is shown again below.

~~~C
if (g_file_load_contents (files[0], NULL, &contents, &length, NULL, NULL)) {
  gtk_text_buffer_set_text (tb, contents, length);
  g_free (contents);
  if ((filename = g_file_get_basename (files[0])) != NULL) {
    gtk_window_set_title (GTK_WINDOW (win), filename);
    g_free (filename);
  }
  gtk_widget_show (win);
} else {
  if ((filename = g_file_get_path (files[0])) != NULL) {
    g_print ("No such file: %s.\n", filename);
    g_free (filename);
  }
  gtk_window_destroy (GTK_WINDOW (win));
}
~~~

The function `g_file_load_contents` loads the file contents into a buffer, which is automatically allocated, and sets `contents` to point the buffer.
And the length of the buffer is set to `length`.
It returns `TRUE` if the file's contents are successfully loaded. `FALSE` if an error happens.

If the function succeeds, it inserts the contents into GtkTextBuffer, frees the buffer memories pointed by `contents`, sets the title of the window,
frees the memories pointed by `filename` and shows the window.
If it fails, it outputs an error message and destroys the window.

## GtkNotebook

GtkNotebook is a container widget that contains multiple children with tabs in it.

![GtkNotebook](../image/screenshot_gtk_notebook.png){width=13.2cm height=5.325cm}

Look at the screenshots above.
The left one is a window at the startup.
It shows the file `pr1.c`.
The filename is in the left tab.
After clicking on the right tab, the contents of `tfv1.c` appears.
It is shown in the right of the screenshot.

GtkNotebook widget is between GtkApplicationWindow and GtkScrolledWindow.
Now I want to show you the program `tfv4.c`.

@@@include
tfv/tfv4.c
@@@

Most of the change is in the function `app_open`.
The numbers at the left of the following items are line numbers in the source code.

- 11-13: Variables `nb`, `lab` and `nbp` are defined and points GtkNotebook, GtkLabel and GtkNotebookPage respectively.
- 23: The window's title is set to "file viewer".
- 25: The size of the window is set to maximum because a big window is appropriate for file viewers.
- 27-28 GtkNotebook is created and inserted to the GtkApplicationWindow as a child.
- 30-59 For-loop. Each loop corresponds to an argument. And files[i] is GFile object with respect to the i-th argument.
- 32-37 GtkScrollledWindow, GtkTextView are created and GtkTextBuffer is get from the GtkTextView.
GtkTextView is connected to GtkScrolledWindow as a child.
They corresponds to each file, so they are created inside the for-loop.
- 39-40 inserts the contents of the file into GtkTextBuffer and frees the memory pointed by `contents`.
- 41-43: Gets the filename and creates GtkLabel with the filename.
Frees `filename`.
- 44-45: If `filename` is NULL, creates GtkLabel with the empty string.
- 46: Appends GtkScrolledWindow and GtkLabel to GtkNotebook.
At the same time, a GtkNoteBookPage widget is created automatically.
The GtkScrolledWindow widget is connected to the GtkNotebookPage.
Therefore, the structure is like this:

~~~
    GtkNotebook -- GtkNotebookPage -- GtkScrolledWindow
~~~

- 47: Gets GtkNotebookPage widget and sets `nbp` to point the GtkNotebookPage.
- 48: GtkNotebookPage has a property "tab-expand".
If it is set to TRUE then the tab expands horizontally as long as possible.
If it is FALSE, then the width of the tab is determined by the size of the label.
`g_object_set` is a general function to set properties in any objects.
See [GObject API Reference, g\_object\_set](https://docs.gtk.org/gobject/method.Object.set.html).
- 49-51: If it fails to read the file, "No such file" message is outputted.
Frees `filename`.
- 52-53: If `filename` is NULL, "No valid file is given" message is outputted.
- 55-58: If at least one file was read, then the number of GtkNotebookPage is greater than zero.
If it's true, it shows the window.
If it's false, it destroys the window.

