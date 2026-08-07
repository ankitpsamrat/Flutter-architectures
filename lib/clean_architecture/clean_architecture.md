# Clean Architecture (Flutter)

Clean Architecture isn't a UI pattern like MVC/MVP/MVVM — it's a **layered dependency-rule architecture** (from Robert C. Martin / "Uncle Bob"). It's usually **combined with** Bloc, Provider, or Riverpod for the presentation layer's state management.

## Core Rule: The Dependency Rule
> Dependencies only point **inward**. Inner layers know nothing about outer layers.

```
 ┌─────────────────────────────────────────┐
 │  Presentation (UI, Bloc/ViewModel)       │
 │  ┌─────────────────────────────────┐    │
 │  │  Domain (Entities, UseCases,     │    │
 │  │  Repository interfaces)          │    │
 │  │  ┌──────────────────────────┐   │    │
 │  │  │  (pure business rules,   │   │    │
 │  │  │   no Flutter, no packages)│  │    │
 │  │  └──────────────────────────┘   │    │
 │  └─────────────────────────────────┘    │
 └─────────────────────────────────────────┘
 Data (Repository implementations, API, DB) -> implements Domain's interfaces
```

## Layers & Responsibilities

### 1. Domain Layer (innermost — the core)
- **Entities**: plain Dart classes representing core business objects (no `@JsonSerializable`, no Flutter imports)
- **Repository interfaces (abstract classes)**: contracts like `abstract class TodoRepository { Future<List<Todo>> getTodos(); }`
- **UseCases** (a.k.a. Interactors): one class per business action, calls repository interface
- **Zero dependencies** on Flutter SDK or any external package — pure Dart

```dart
// domain/entities/todo.dart
class Todo {
  final String id;
  final String title;
  final bool isCompleted;
  const Todo({required this.id, required this.title, this.isCompleted = false});
}

// domain/repositories/todo_repository.dart
abstract class TodoRepository {
  Future<List<Todo>> getTodos();
  Future<void> addTodo(Todo todo);
}

// domain/usecases/get_todos_usecase.dart
class GetTodosUseCase {
  final TodoRepository repository;
  GetTodosUseCase(this.repository);

  Future<List<Todo>> call() => repository.getTodos();
}
```

### 2. Data Layer
- **Models**: extend/implement domain entities, add `fromJson`/`toJson`
- **Repository implementations**: implement the domain's abstract repository, decide whether to hit API or local cache
- **Data sources**: `RemoteDataSource` (API/Dio/http), `LocalDataSource` (Hive/SharedPreferences/SQLite)

```dart
// data/models/todo_model.dart
class TodoModel extends Todo {
  const TodoModel({required super.id, required super.title, super.isCompleted});

  factory TodoModel.fromJson(Map<String, dynamic> json) => TodoModel(
        id: json['id'],
        title: json['title'],
        isCompleted: json['isCompleted'] ?? false,
      );
}

// data/repositories/todo_repository_impl.dart
class TodoRepositoryImpl implements TodoRepository {
  final TodoRemoteDataSource remoteDataSource;
  TodoRepositoryImpl(this.remoteDataSource);

  @override
  Future<List<Todo>> getTodos() => remoteDataSource.fetchTodos();

  @override
  Future<void> addTodo(Todo todo) => remoteDataSource.addTodo(todo);
}
```

### 3. Presentation Layer (outermost)
- Bloc/Cubit/ViewModel + Widgets
- Calls **UseCases only** — never touches Repository or DataSource directly
- This is where you plug in Bloc (see next file) or MVVM

```dart
// presentation/bloc/todo_bloc.dart
class TodoBloc extends Bloc<TodoEvent, TodoState> {
  final GetTodosUseCase getTodosUseCase;

  TodoBloc(this.getTodosUseCase) : super(const TodoInitial()) {
    on<LoadTodosEvent>((event, emit) async {
      final todos = await getTodosUseCase();
      emit(TodoLoaded(todos));
    });
  }
}
```

### Dependency Injection wiring (typically with `get_it`)
```dart
final getIt = GetIt.instance;

void setupLocator() {
  getIt.registerLazySingleton<TodoRemoteDataSource>(() => TodoRemoteDataSourceImpl());
  getIt.registerLazySingleton<TodoRepository>(() => TodoRepositoryImpl(getIt()));
  getIt.registerLazySingleton(() => GetTodosUseCase(getIt()));
  getIt.registerFactory(() => TodoBloc(getIt()));
}
```

## Folder structure (typical)
```
lib/
 ├── domain/
 │   ├── entities/
 │   ├── repositories/      (abstract)
 │   └── usecases/
 ├── data/
 │   ├── models/
 │   ├── datasources/
 │   └── repositories/      (implementations)
 └── presentation/
     ├── bloc/ (or viewmodel/)
     ├── pages/
     └── widgets/
```

## Why UseCases specifically (common interview question)
- Keeps Bloc/Presenter thin — it just calls `usecase()`, doesn't know about API/DB details
- Each UseCase = one business action = easy to unit test in isolation
- If business rule changes (e.g. "todos older than 30 days auto-archive"), it lives in ONE place (UseCase), not scattered across Blocs

## Pros
- Highly testable — every layer can be tested independently with mocks
- Framework-independent domain layer (could theoretically reuse business logic outside Flutter)
- Scales very well for large teams/large codebases
- Clear separation of concerns — easy to onboard new devs once structure is understood

## Cons
- Significant boilerplate for small/medium apps (entities, models, interfaces, impls, usecases — for a single todo feature)
- Overkill for simple apps/MVPs
- Steeper learning curve

## When to use
- Large production apps, apps expected to scale/maintain long-term, team projects
- Almost always paired with Bloc (very common combo: "Clean Architecture + Bloc") in interviews
