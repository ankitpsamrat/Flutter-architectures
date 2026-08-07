# BLoC (Business Logic Component) Architecture

BLoC is both a **pattern** and a **package** (`flutter_bloc`). As a pattern, it's about separating business logic from UI using **Streams** and unidirectional data flow. As a package, it gives you `Bloc`/`Cubit` classes to implement that pattern easily.

## Flow
```
User -> View -> Event -> Bloc -> State -> View (rebuilds)
```

Strictly **unidirectional**: View never mutates state directly, never calls business logic methods directly (for `Bloc`) — it only dispatches Events. Bloc processes Events, decides new State, emits it. View listens and rebuilds.

## Responsibilities

**Event**
- Describes **what happened** (user action or trigger) — e.g. `AddTodoEvent`, `LoadTodosEvent`
- Input to the Bloc

**State**
- Describes **what the UI should look like** at a point in time — e.g. `TodoLoading`, `TodoLoaded`, `TodoError`
- Output from the Bloc, immutable, compared with `Equatable`

**Bloc**
- Receives Events via `on<EventType>(handler)`
- Runs business logic (often by calling a UseCase/Repository)
- Emits new States via `emit()`
- Never directly touches widgets — no `BuildContext`, no Flutter UI imports

## Example (from earlier Todo project)
```dart
sealed class TodoEvent extends Equatable {
  const TodoEvent();
  @override
  List<Object?> get props => [];
}
final class AddTodoEvent extends TodoEvent {
  final String title;
  const AddTodoEvent(this.title);
  @override
  List<Object?> get props => [title];
}

sealed class TodoState extends Equatable {
  const TodoState();
  @override
  List<Object?> get props => [];
}
final class TodoLoaded extends TodoState {
  final List<Todo> todos;
  const TodoLoaded(this.todos);
  @override
  List<Object?> get props => [todos];
}

class TodoBloc extends Bloc<TodoEvent, TodoState> {
  TodoBloc() : super(const TodoInitial()) {
    on<AddTodoEvent>((event, emit) {
      final updated = [...currentTodos, Todo(id: '1', title: event.title)];
      emit(TodoLoaded(updated));
    });
  }
}

// View
context.read<TodoBloc>().add(AddTodoEvent('Buy milk')); // dispatch event
BlocBuilder<TodoBloc, TodoState>(
  builder: (context, state) => /* render based on state */,
)
```

## Bloc vs Cubit
| | Bloc | Cubit |
|---|---|---|
| Trigger | `add(Event())` | direct method call |
| Traceability | Full event → state history (great for logging/analytics/`BlocObserver`) | Only tracks state changes, not "why" |
| Boilerplate | More (Event classes + handlers) | Less (just methods) |
| Concurrency control | Built-in via `bloc_concurrency` transformers (`droppable`, `restartable`, `sequential`, `concurrent`) | Not applicable — it's just async methods |
| Best for | Complex flows, multi-step processes, audit/undo-redo needs | Simple-medium state, quick implementation |

## Key building blocks

### BlocObserver — global logging (very commonly asked)
```dart
class MyBlocObserver extends BlocObserver {
  @override
  void onTransition(Bloc bloc, Transition transition) {
    super.onTransition(bloc, transition);
    print(transition); // logs every event -> state change, app-wide
  }

  @override
  void onError(BlocBase bloc, Object error, StackTrace stackTrace) {
    super.onError(bloc, error, stackTrace);
    print('$error');
  }
}

void main() {
  Bloc.observer = MyBlocObserver();
  runApp(MyApp());
}
```

### Concurrency transformers (`bloc_concurrency` package)
Handles what happens if events fire faster than they're processed — a favorite interview topic:
```dart
on<SearchEvent>(_onSearch, transformer: restartable());
// droppable()   -> ignores new events while one is processing
// restartable()  -> cancels current processing, starts fresh with latest event (great for search-as-you-type)
// sequential()   -> processes one at a time in order (default behavior)
// concurrent()   -> processes all events in parallel
```

### Bloc-to-Bloc communication
```dart
class TodoBloc extends Bloc<TodoEvent, TodoState> {
  final AuthBloc authBloc;
  late final StreamSubscription authSubscription;

  TodoBloc(this.authBloc) : super(const TodoInitial()) {
    authSubscription = authBloc.stream.listen((authState) {
      if (authState is LoggedOut) add(ClearTodosEvent());
    });
  }

  @override
  Future<void> close() {
    authSubscription.cancel();
    return super.close();
  }
}
```

### HydratedBloc — persisting state automatically
```dart
class TodoBloc extends HydratedBloc<TodoEvent, TodoState> {
  @override
  TodoState? fromJson(Map<String, dynamic> json) => TodoLoaded.fromJson(json);
  @override
  Map<String, dynamic>? toJson(TodoState state) =>
      state is TodoLoaded ? state.toJson() : null;
}
```

## Bloc + Clean Architecture (the standard combo)
Bloc sits in the **Presentation layer**, calls **UseCases** from the Domain layer — see `04-clean-architecture.md`. This combo (Clean Architecture + Bloc) is the most commonly expected setup in mid/senior Flutter interviews.

## Pros
- Fully unidirectional, highly predictable data flow
- Excellent traceability via events (debugging, analytics, `BlocObserver`)
- First-class testing support (`bloc_test`, `mocktail`)
- Battle-tested at scale, huge community, well-documented

## Cons
- More boilerplate than Provider/Riverpod/MVVM (Event + State classes for every feature)
- Steeper learning curve for beginners (streams, sealed classes, emit rules)
- Overkill for trivial screens (e.g. a single toggle switch)

## When to use
- Medium-large apps needing predictable, testable, traceable state management
- Teams that value strict separation and enforce architecture consistency
- The most commonly expected state management answer in Flutter interviews today
