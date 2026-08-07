# MVP (Model-View-Presenter)

## Flow
```
User -> View -> Presenter -> Model -> Presenter -> View
```

Unlike MVC, the **View and Presenter reference each other directly** (usually via an interface/contract), and the **View is "dumb"** — it never talks to Model directly.

## Responsibilities

**Model**
- API calls, database, business data
- Same as MVC — no knowledge of View/Presenter

**View**
- UI only, and is defined by a **contract (interface)**
- Forwards user actions to Presenter
- Exposes methods Presenter can call to update UI (e.g. `showLoading()`, `showTodos(list)`, `showError(msg)`)
- Contains **no logic** — not even simple decisions

**Presenter**
- Holds a reference to the View (via interface) and the Model
- Contains ALL UI logic/decisions
- Calls Model, gets data, then explicitly calls View's methods to render it
- Fully unit-testable since it doesn't depend on Flutter widgets — View is mocked via its interface

## Example structure

```dart
// Contract
abstract class TodoView {
  void showLoading();
  void showTodos(List<Todo> todos);
  void showError(String message);
}

// Presenter — pure Dart, no Flutter imports, fully testable
class TodoPresenter {
  final TodoView view;
  final TodoRepository repository;

  TodoPresenter(this.view, this.repository);

  Future<void> loadTodos() async {
    view.showLoading();
    try {
      final todos = await repository.fetchTodos();
      view.showTodos(todos);
    } catch (e) {
      view.showError(e.toString());
    }
  }
}

// View — implements the contract
class TodoPage extends StatefulWidget {
  @override
  State<TodoPage> createState() => _TodoPageState();
}

class _TodoPageState extends State<TodoPage> implements TodoView {
  late final TodoPresenter presenter;
  List<Todo> todos = [];
  bool isLoading = false;
  String? error;

  @override
  void initState() {
    super.initState();
    presenter = TodoPresenter(this, TodoRepository());
    presenter.loadTodos();
  }

  @override
  void showLoading() => setState(() => isLoading = true);

  @override
  void showTodos(List<Todo> newTodos) => setState(() {
        isLoading = false;
        todos = newTodos;
      });

  @override
  void showError(String message) => setState(() {
        isLoading = false;
        error = message;
      });

  @override
  Widget build(BuildContext context) {
    // purely renders based on current fields — no logic
    if (isLoading) return const CircularProgressIndicator();
    if (error != null) return Text(error!);
    return ListView(children: todos.map((t) => Text(t.title)).toList());
  }
}
```

## Pros
- Presenter is 100% unit-testable (no Flutter/widget dependency)
- Clear separation — View truly has no logic
- Good middle ground between MVC's messiness and MVVM/Clean's complexity

## Cons
- Presenter holds a direct reference to View → tight coupling (need to mock the interface in tests)
- Verbose — lots of interface methods to define and implement for every screen
- No built-in reactive stream/state broadcasting like MVVM or Bloc — updates are manual method calls
- Less popular in modern Flutter (MVVM/Bloc dominate)

## When to use
- Medium apps where you want testability without adopting a state-management package
- Rare in modern Flutter codebases — mostly seen in apps ported from native Android (where MVP was very common)