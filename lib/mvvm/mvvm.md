# MVVM (Model-View-ViewModel)

## Flow
```
User -> View -> ViewModel -> Model
                    |
                    v
         (View listens/reacts to ViewModel's
          observable state automatically)
```

The key difference from MVP: **View does NOT hold a reference to ViewModel's methods being called on it.** Instead, View **observes** ViewModel's state (via `ChangeNotifier`, `ValueNotifier`, streams, or Riverpod/Provider) and rebuilds automatically when state changes. No manual `showX()` calls.

## Responsibilities

**Model**
- API calls, database, business data — same role as always

**View**
- UI only
- Binds/subscribes to ViewModel's exposed observable properties
- Forwards user actions to ViewModel (calls its methods)
- Rebuilds reactively when observed state changes — no direct method calls from ViewModel to View

**ViewModel**
- Exposes state as observable properties (`ChangeNotifier`, streams, `ValueNotifier`)
- Contains UI logic and calls Model
- Has **no reference to View at all** (unlike Presenter in MVP) — doesn't know Flutter/widgets exist
- View "listens" to it; ViewModel never calls View directly

## Example (using ChangeNotifier + Provider)

```dart
// ViewModel — no Flutter widget dependency, no reference to View
class TodoViewModel extends ChangeNotifier {
  final TodoRepository repository;
  TodoViewModel(this.repository);

  List<Todo> todos = [];
  bool isLoading = false;
  String? error;

  Future<void> loadTodos() async {
    isLoading = true;
    notifyListeners(); // View observes this and rebuilds

    try {
      todos = await repository.fetchTodos();
      error = null;
    } catch (e) {
      error = e.toString();
    } finally {
      isLoading = false;
      notifyListeners();
    }
  }

  void addTodo(String title) {
    todos = [...todos, Todo(id: DateTime.now().toString(), title: title)];
    notifyListeners();
  }
}

// View — observes ViewModel via Consumer/context.watch, doesn't get "told" what to render
class TodoPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (_) => TodoViewModel(TodoRepository())..loadTodos(),
      child: Consumer<TodoViewModel>(
        builder: (context, vm, child) {
          if (vm.isLoading) return const CircularProgressIndicator();
          if (vm.error != null) return Text(vm.error!);
          return ListView(
            children: vm.todos.map((t) => Text(t.title)).toList(),
          );
        },
      ),
    );
  }
}
```

## MVVM vs MVP — the core distinction (common interview question)
| | MVP | MVVM |
|---|---|---|
| View <-> mediator link | Presenter holds reference to View, calls its methods directly | ViewModel has **no** reference to View — View observes ViewModel |
| Update mechanism | Manual (`view.showTodos(list)`) | Reactive/automatic (`notifyListeners()`, streams) |
| Testability | High (mock View interface) | High (ViewModel is plain Dart, test state directly) |
| Coupling | Tighter (Presenter ↔ View interface) | Looser (ViewModel is fully independent) |

## Pros
- ViewModel is fully independent of Flutter — very testable
- Reactive updates reduce boilerplate compared to MVP's manual method calls
- Popular pairing with `Provider`, `Riverpod`, or `get_it` for DI
- Scales well for medium-large apps

## Cons
- If using plain `ChangeNotifier`, no built-in event history/traceability (unlike Bloc)
- Easy to accidentally leak business logic into View if not disciplined
- Multiple ways to implement it (ChangeNotifier vs Riverpod vs signals) → less standardized than Bloc

## When to use
- Medium-large apps wanting reactive architecture without full Bloc boilerplate
- Very common with `Provider` and `Riverpod` in real-world Flutter apps
