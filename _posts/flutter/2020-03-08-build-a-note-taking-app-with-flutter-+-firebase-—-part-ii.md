---
layout: post
title: "Build a note-taking app with Flutter + Firebase — Part II"
description: "Build a note-taking app with Flutter + Firebase — Part II"
category: flutter
tags:
path: _posts/flutter/2020-03-08-build-a-note-taking-app-with-flutter-+-firebase-—-part-ii.md
subtitle: Implements a plain-text editor with reversible actions & Hero transitions
feature: /images/feature-flutter-keep-02.jpg
category: [flutter, firebase, firestore, dart, flutter-web, note-taking]
tags:
excerpt: |
  This is the second story of the series: "Build a note-taking app with Flutter + Firebase".
  In part I, we've built the first screen for the notebook app, Flutter Keep.
  In this article, we're going to create a note editor, with reversible operations supported, and explore the magical Hero animations.
---

![head image](/images/feature-flutter-keep-02.jpg)


> 这篇原创文章最初发表在Medium上的 [Flutter Community][medium story] publication中。<br>
> This article has been published originally in the [Flutter Community][medium story] on Medium.


Nice to have you in this second part of the series of ***Build a note-taking app with Flutter + Firebase***. If you haven't read the previous article, please find it [here][part I].

In [part I], we've built the first screen for the notebook app, ***Flutter Keep***. In this article, we're going to create a note editor, with reversible operations supported, and explore the magical *[Hero][hero animation]* animations.

---

## The note editor

There're many types of notes in [Google Keep], including plain-text notes, audio notes, and checklists, with optional image attachments. However, in this example, we'll focus on the plain-text editor to keep things simple.

The following is a preview of what we're going to build:

![Note editor preview](/images/fltkeep-note-editor-preview.jpg)

In essence, it composites of two text fields, one for the title, another for the plain-text content.
Plus, a top `AppBar` and a `ModalBottomSheet` provides actions to update either the state or the color of the note.

So let's start with a `StatefulWidget`:

```dart
// The editor of a [Note], also shows every detail about a single note.
class NoteEditor extends StatefulWidget {
  /// Create a [NoteEditor],
  /// provides an existed [note] in edit mode, or `null` to create a new one.
  const NoteEditor({Key key, this.note}) : super(key: key);

  final Note note;

  @override
  State<StatefulWidget> createState() => _NoteEditorState(note);
}

class _NoteEditorState extends State<NoteEditor> {
  _NoteEditorState(Note note)
    : this._note = note ?? Note(),
    _originNote = note?.copy() ?? Note(),
    this._titleTextController = TextEditingController(text: note?.title),
    this._contentTextController = TextEditingController(text: note?.content);
  ...

  /// Returns `true` if the note is modified.
  bool get _isDirty => _note != _originNote;
  ...
}
```


In the editor state, we keep a copy of the original note (could be an empty one) to check whether the editor is dirty.

And don't forget to make it accessible from the `HomeScreen`:

```dart
/// Presses the FAB to create a new note
Widget _fab(BuildContext context) => FloatingActionButton(
  ...
  onPressed: () => Navigator.pushNamed(context, '/note'),
);

/// Starts editing a note when tapped
void _onNoteTap(Note note) =>
  Navigator.pushNamed(context, '/note', arguments: { 'note': note });
```

---

Ok, go on building the widget:

```dart
@override
Widget build(BuildContext context) {
  final uid = Provider.of<CurrentUser>(context).data.uid;
  return ChangeNotifierProvider.value(
    value: _note,
    child: Consumer<Note>(
      builder: (_, __, ___) => Theme(
        data: Theme.of(context).copyWith(
          primaryColor: _noteColor,
          ...
        ),
        child: AnnotatedRegion<SystemUiOverlayStyle>(
          // tint the Android system navigation bar
          value: SystemUiOverlayStyle.dark.copyWith(
            statusBarColor: _noteColor,
            systemNavigationBarColor: _noteColor,
            systemNavigationBarIconBrightness: Brightness.dark,
          ),
          child: Scaffold(
            key: _scaffoldKey,
            appBar: AppBar(
              actions: _buildTopActions(context, uid),
            ),
            body: WillPopScope(
              onWillPop: () => _onPop(uid),
              child: _buildBody(context, uid), // textfields for title/content
            ),
            bottomNavigationBar: _buildBottomAppBar(context),
          ),
        ),
      ),
    ),
  );
}
```

We're going to save the note to FireStore before the editor is closed, by using a `WillPopScope` widget, which fires the `onWillPop` callback right before the current screen is being dismissed.

```dart
/// Callback before the user dismiss the editor
Future<dynamic> _onPop(String uid) => _isDirty
  ? saveToFireStore(uid, _note)
  : Future.value();
```

Introducing a `ChangeNotifierProvider` is to provide the `Note` object being edited to descendant widgets to keep them aligned with the latest state. For example, the editor should update the background color when the user picks a different one in the `ColorPicker` widget.

To use `ChangeNotifierProvider`, the `Note` class has to extend the `ChangeNotifier`, and to `notifyListeners` whenever its state has changed:

```dart
class Note extends ChangeNotifier {
  ...
  /// Update specified properties and notify the listeners
  void updateWith({
    String title,
    String content,
    Color color,
    NoteState state,
  }) {
    ...
    notifyListeners();
  }
}
```

We'll see how it works.

---

Actions like archiving and picking a theme color are organized in a bottom sheet.
One thing that may be confusing is when we `showModalBottomSheet`, we have to provide the note object again, or the descendant widgets (of the bottom sheet) won't be able to retrieve it via `Consumer` or `Provider.of`. (It's a different widget tree from the editor body)

```dart
showModalBottomSheet(
  context: context,
  backgroundColor: _noteColor,
  builder: (context) => ChangeNotifierProvider.value(
    value: _note,
    child: Consumer<Note>(
      builder: (_, note, __) => Container(
        color: note.color ?? kDefaultNoteColor, // use the latest picked color
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: <Widget>[
            NoteActions(), // list of available actions: archiving, deletion...
            LinearColorPicker(), // a horizontal color list
          ],
        ),
      ),
    ),
  ),
);
```

To understand how a `ChangeNotifierProvider` works, we take the color picker as an example.

What does the `LinearColorPicker` do? It renders a horizontal list of available tints for a note:

```dart
class LinearColorPicker extends StatelessWidget {
  Color _currColor(Note note) => note?.color ?? kDefaultNoteColor;

  @override
  Widget build(BuildContext context) {
    Note note = Provider.of<Note>(context);
    return SingleChildScrollView(
      scrollDirection: Axis.horizontal,
      child: Row(
        children: kNoteColors.map((color) => InkWell(
          child: Container(
            ...
            color: color,
            child: color == _currColor(note) ? const Icon(Icons.check) : null,
          ),
          onTap: () {
            if (color != _currColor(note)) {
              note.updateWith(color: color);
            }
          },
        )).toList(),
      ),
    );
  }
}
```

When one of the tints is selected, the picker updates the `color` property of the note, and that is it.
An ancestor widget (of the `ColorPicker`) we've built previously, which watching the note, gets notified and then refreshes the editor screen so that we'll see the whole editor is tinted with the color we pick.

![Editor state synchronization](/images/fltkeep-editor-sync.gif)

The other actions, such as deletion and archiving, share the same mechanism.

---

## Reversible operations

We can now edit a note by updating the properties including the state. But how about the UX? What if users delete a note by accident?

For dangerous operations like deleting or archiving, a `SnackBar` could be used to provide a reverse action, e.g., restoring or unarchiving, in addition to a prompt message.

![SnackBar with an undo action](/images/fltkeep-snackbar.gif)

To implement reversible operations more cleanly, we're going to apply the [***Command Pattern***].

First, we define the command interface, which is responsible for applying an action to a note.

```dart
abstract class NoteCommand {
  final String id;
  final String uid;

  /// Defines an reversible action to a note, provides the note [id], and the current user's [uid].
  const NoteCommand({
    @required this.id,
    @required this.uid,
  });

  /// Returns message about the result of the action.
  String get message => '';

  /// Executes this command.
  Future<void> execute();

  /// Undo this command.
  Future<void> revert();
}
```

For this notebook app, actions considered reversible are all about mutating the state of a note. So only one concrete command is needed. However, nothing prevents you from extending it to other situations.

```dart
class NoteStateUpdateCommand extends NoteCommand {
  final NoteState from;
  final NoteState to;

  /// Create a [NoteCommand] to update state of a note [from] the current state [to] another.
  NoteStateUpdateCommand({
    @required String id,
    @required String uid,
    @required this.from,
    @required this.to,
  }) : super(id: id, uid: uid);

  @override
  String get message {
    switch (to) {
      case NoteState.deleted:
        return 'Note moved to trash';
      case NoteState.archived:
        return 'Note archived';
      ...
    }
  }

  @override
  Future<void> execute() => updateNoteState(to, id, uid);

  @override
  Future<void> revert() => updateNoteState(from, id, uid);
}
```

Next, we produce and consume commands, for example:

```dart
// when pressed, dissmiss the bottom sheet and return a command
IconButton(
  icon: const Icon(Icons.archive),
  tooltip: 'Archive',
  onPressed: () => Navigator.pop(context, NoteStateUpdateCommand(
    id: _note.id,
    uid: uid,
    from: _note.state, // the current state
    to: NoteState.archived, // the transition target
  )),
),

// receiving a command
void _showNoteBottomSheet(BuildContext context) async {
  final command = await showModalBottomSheet<NoteCommand>(
    ...
  );

  if (command != null) {
    processNoteCommand(_scaffoldKey.currentState, command);
  }
}

/// Processes the given [command], displays a `SnackBar`.
Future<void> processNoteCommand(ScaffoldState scaffoldState, NoteCommand command) async {
  await command.execute();
  final msg = command.message;
  if (mounted && msg?.isNotEmpty == true) {
    scaffoldState?.showSnackBar(SnackBar(
      content: Text(msg),
      action: SnackBarAction(
        label: 'Undo',
        onPressed: () => command.revert(),
      ),
    ));
  }
}
```

Done, we've made the dangerous operations reversible!

---

## Hero transition

Now we have a working note editor, let's take a step further, by adding a beautiful transition animation between the `HomeScreen` and the `NoteEditor`.

![Hero animation demo](/images/fltkeep-hero-transition.gif)

We can see that a note item grows to the size of the entire screen, from where it located in the grid. In Flutter, that's called a [Hero Animation].

The usage is simple. First, we wrap the note item in the grid or list with a `Hero` widget:

```dart
// in the note list
@override
Widget build(BuildContext context) => Hero(
  tag: 'NoteItem${note.id}',
  child: DefaultTextStyle(
    style: kNoteTextLight,
    child: Container(
      ...
    ),
  ),
);
```

Then wrap the editor widget too:

```dart
@override
Widget build(BuildContext context) {
  final uid = Provider.of<CurrentUser>(context).data.uid;
  return ChangeNotifierProvider.value(
    value: _note,
    child: Consumer<Note>(
      builder: (_, __, ___) => Hero(
        tag: 'NoteItem${_note.id}',
        child: DefaultTextStyle(
          style: ...
```

In the above snippets, the two tags passed to the `Hero` widgets must be identical.

And the `DefaultTextStyle` widgets are applied to avoid the big underlined text during screen transition on the iOS platform.
Now we can leave the rest to the Flutter framework.

What noticeable is that the standard screen transition animation is platform-specific. You could make a custom transition as you need, but it is beyond the scope of this article. Please refer to this [cookbook][custom transition].

---

## Tips

In an example app, we don't bother to apply patterns like **[BLOC]**. But there are still ways to keep the code clean and avoid boilerplates.

### Mixins

For example, we can move the reversible operation handling procedure to a stand-alone **[mixin]**, to make a cleaner separation between UI and logic, and also make it reusable.

```dart
mixin CommandHandler<T extends StatefulWidget> on State<T> {
  /// Processes the given [command].
  Future<void> processNoteCommand(ScaffoldState scaffoldState, NoteCommand command) async {
    if (command == null) {
      return;
    }
    await command.execute();
    final msg = command.message;
    if (mounted && msg?.isNotEmpty == true) {
      scaffoldState?.showSnackBar(SnackBar(
        content: Text(msg),
        action: SnackBarAction(
          label: 'Undo',
          onPressed: () => command.revert(),
        ),
      ));
    }
  }
}
```

Whenever you need to handle commands, just mix it in:

```dart
class _HomeScreenState extends State<HomeScreen> with CommandHandler {
  Widget _fab(BuildContext context) => FloatingActionButton(
    child: const Icon(Icons.add),
    onPressed: () async {
      final command = await Navigator.pushNamed(context, '/note');
      processNoteCommand(_scaffoldKey.currentState, command);
    },
  );
  ...
}
```

### Extension methods

In addition, with Dart SDK `2.7.0` or later, we can leverage [extension methods] to do things magical.

We can augment the Note model with FireStore related functionalities:

```dart
extension NoteStore on Note {
  /// Save this note to FireStore.
  Future<dynamic> saveToFireStore(String uid) async {
    final col = notesCollection(uid);
    return id == null
      ? col.add(toJson())
      : col.document(id).updateData(toJson());
  }
	...
}
```

Which makes persisting a note as easy as a method call:

```dart
/// Save the note before the editor is dismissed
Future<dynamic> _onPop(String uid) => _isDirty
  ? _note.saveToFireStore(uid)
  : Future.value();
```

We can even add properties and methods to an enumeration, which is impossible in the declaration of enumerated types:

```dart
extension NoteStateX on NoteState {
  /// Checks if a note in this state can be edited.
  bool get canEdit => this < NoteState.deleted;

	/// Returns true if this state is preceding to the other one.
  bool operator <(NoteState other) => (this?.index ?? 0) < (other?.index ?? 0);

  /// Message describes the state transition.
  String get message {
    switch (this) {
      case NoteState.deleted:
        return 'Note moved to trash';
			...
    }
  }
  ...
}
```

That saves us a lot of repeated code!

---

Wrapping it up, we've delivered a working plain-text note editor in this iteration. We've even added features like reversible actions and Hero transitions. Please find the complete code example in this [GitHub repo].

In the next part, I'd like to introduce how to query different subsets of notes from FireStore, and the issue of composite indexes.

Thank you for reading! 🙌


> 这篇原创文章最初发表在Medium上的 [Flutter Community][medium story] publication中。<br>
> This article has been published originally in the [Flutter Community][medium story] on Medium.


[part I]: /build-a-note-taking-app-with-flutter-firebase-part-i/
[Hero animation]: https://flutter.dev/docs/development/ui/animations/hero-animations
[Google Keep]: https://www.google.com/keep
[setSystemUIOverlayStyle]: https://api.flutter.dev/flutter/services/SystemChrome/setSystemUIOverlayStyle.html
[Command Pattern]: https://en.wikipedia.org/wiki/Command_pattern
[custom transition]: https://flutter.dev/docs/cookbook/animation/page-route-animation
[BLOC]: https://bloclibrary.dev/
[mixin]: https://dart.dev/guides/language/language-tour#adding-features-to-a-class-mixins
[extension methods]: https://dart.dev/guides/language/extension-methods
[GitHub repo]: https://github.com/xinthink/flutter-keep
[medium story]: https://medium.com/flutter-community/build-a-note-taking-app-with-flutter-firebase-part-ii-73c1aa6b346
