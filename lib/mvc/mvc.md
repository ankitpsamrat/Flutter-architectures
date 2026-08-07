# MVC (Model-View-Controller)

## Flow
```
User -> View -> Controller -> Model
                    ^            |
                    |____________|
                 (Controller updates View)
```

**Corrected flow:** Model never talks back to Controller in a "push" sense inside Flutter apps — Controller *pulls* data from Model, then Controller updates View directly. Model does not know Controller or View exist.

```
User interacts with View
   -> View notifies Controller
   -> Controller calls Model (API/DB)
   -> Model returns data to Controller
   -> Controller updates View
```

## Responsibilities

**Model**
- API calls, database, business data
- Plain data classes + repositories
- Has zero knowledge of View or Controller

**View**
- UI only — widgets, layout, styling
- Should NOT contain business logic
- Notifies Controller of user actions (button press, text input)

**Controller**
- Handles user input events (button clicks, form submit)
- Calls Model to fetch/update data
- Updates View once data is ready

## Flutter-specific reality (important for interviews)

In classic MVC, View and Controller are separate. In Flutter, this separation is **hard to enforce** because:
- A `StatefulWidget`'s `State` class often plays **both View and Controller roles** — it builds UI (View) *and* handles `onPressed` logic, calls APIs, and calls `setState()` (Controller).
- This tight coupling is the #1 criticism of MVC in Flutter — it leads to bloated `State` classes ("god widgets").

```dart
class CounterPage extends StatefulWidget {
  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int count = 0; // Model-ish data, sitting inside "Controller"

  void increment() { // Controller logic
    setState(() => count++); // updates View
  }

  @override
  Widget build(BuildContext context) { // View
    return ElevatedButton(
      onPressed: increment,
      child: Text('$count'),
    );
  }
}
```

## Pros
- Simple, minimal boilerplate
- Easy to understand for small apps
- No extra packages needed

## Cons
- View and Controller tend to merge in Flutter (StatefulWidget does both jobs)
- Hard to unit test Controller logic in isolation (it's tangled with widget lifecycle)
- Doesn't scale well for medium/large apps
- No formal state management — relies on `setState()`

## When to use
- Very small apps, prototypes, learning projects
- Not recommended for production apps with growing complexity