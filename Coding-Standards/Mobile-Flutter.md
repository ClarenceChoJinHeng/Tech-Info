[🏠 Home](../README.md) › [Coding Standards](README.md) › **Mobile — Flutter + Dart**

# Mobile — Flutter + Dart

> Read **[General Principles](General-Principles.md)** first — this page only covers what's specific to Flutter.
> **Which tools we use, why, and how to set them up:** [Tech Stack → Mobile Stack Alternative](../Tech-Stack/Mobile-Stack-Alternative.md). This page is how to write code with them day to day.

> **Scope:** Flutter is the *alternative* mobile stack, used for UI-heavy and animation-heavy apps or where a client mandates it — see [Mobile Stack](../Tech-Stack/Mobile-Stack.md) for when to choose it. Default mobile projects follow [React Native](Mobile-React-Native.md). We have no shipped Flutter project yet, so treat this page as a well-sourced starting point the first project should confirm and correct — not as settled house style.

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Build methods](#build-runs-constantly) | **`build()` is cheap and side-effect free**<br/><sub>It runs on every frame that affects the widget. Fetching belongs in a provider.</sub> |
| [Folder structure](#folder-structure) | **Feature-first: `data`, `domain`, `providers`, `presentation`**<br/><sub>Deleting a feature means deleting a folder.</sub> |
| [Splitting widgets](#split-widgets-into-widgets-not-methods) | **Extract private widget classes, not `_buildX()` methods**<br/><sub>A real widget can be `const` and rebuilds on its own.</sub> |
| [Const constructors](#const-constructors) | **`const` wherever the analyzer allows**<br/><sub>Built once and reused. The cheapest performance win in Flutter.</sub> |
| [Long lists](#long-lists-use-listview-builder) | **`ListView.builder` for anything driven by API data**<br/><sub>Builds only what's visible instead of every row up front.</sub> |
| [Providers](#riverpod-providers) | **`ref.watch` in `build`, `ref.read` in callbacks**<br/><sub>`watch` subscribes to changes; `read` takes the value once.</sub> |
| [Async gaps](#buildcontext-and-async-gaps) | **Check `context.mounted` after every `await`**<br/><sub>The widget may be gone by the time the future finishes.</sub> |
| [Logic placement](#keep-logic-out-of-widgets) | **Widgets render, providers and services do the work**<br/><sub>Logic outside widgets can be tested without pumping a widget tree.</sub> |
| [Theme](#theme-values) | **Colours and text styles from `Theme.of(context)`**<br/><sub>Hard-coded colours ignore dark mode and rebrands.</sub> |
| [Calling the API](#calling-the-api) | **One configured `Dio` instance from `core/`**<br/><sub>Base URL, token and 401 handling are set up once.</sub> |
| [Storage](#storing-tokens-and-data) | **Tokens in `flutter_secure_storage`, everything else in `hive_ce`**<br/><sub>Hive and `shared_preferences` are unencrypted.</sub> |
| [Null safety](#null-safety) | **Handle the null, don't silence it with `!`**<br/><sub>A wrong `!` is a production crash.</sub> |
| [Linting and formatting](#linting-and-formatting) | **`dart format` + `flutter analyze` before every commit**<br/><sub>Both ship in the SDK. Only the config is ours to set.</sub> |
| [Testing](#testing) | **Unit tests for `domain/`, widget tests for shared components**<br/><sub>Widget tests are cheap in Flutter, so write them from the first sprint.</sub> |

* * *

## Concepts you'll meet on this codebase

If you're coming from React, most of this maps over. The names are different and one rule genuinely isn't.

### Everything is a widget

Layout, padding, a button, the whole screen — all widgets, nested into a tree. The React equivalent of a component, with one important difference below.

### Build runs constantly

A widget's `build()` is called on every frame that affects it — which during an animation is 60 times a second. This is the rule that catches React developers out: `build()` must be **cheap and side-effect free**. No API calls, no file reads, no sorting a 500-item list.

```dart
// ❌ Fires a request on every rebuild
@override
Widget build(BuildContext context) {
  final invoices = api.fetchInvoices(); // no
  return InvoiceList(invoices: invoices);
}

// ✅ The provider owns the fetch and caches it; build just reads the result
@override
Widget build(BuildContext context, WidgetRef ref) {
  final invoices = ref.watch(invoiceListProvider);
  return invoices.when(
    data: (list) => InvoiceList(invoices: list),
    loading: () => const LoadingIndicator(),
    error: (e, _) => ErrorView(error: e),
  );
}
```

`.when` forcing you to write all three branches is a feature: loading and error states stop being things you forget.

### Const constructors

A `const` widget is built once and reused on every subsequent rebuild instead of being recreated. It's the cheapest performance win in Flutter and it's mostly free — turn on the `prefer_const_constructors` lint and let the analyzer tell you where.

```dart
// ❌ A new Padding and Text are created on every rebuild
Padding(padding: EdgeInsets.all(16), child: Text('No invoices yet'))

// ✅ Created once, reused every time
const Padding(padding: EdgeInsets.all(16), child: Text('No invoices yet'))
```

### BuildContext and async gaps

A widget's handle on its position in the tree — used to read the theme, navigate, or show a snackbar. The rule that bites: **don't use a `context` across an `await`.** The widget may be gone by the time the future resolves.

```dart
// ❌ Crashes if the user navigated away while this was in flight
await saveInvoice();
Navigator.of(context).pop();

// ✅
await saveInvoice();
if (!context.mounted) return;
Navigator.of(context).pop();
```

### Riverpod providers

A provider is a named piece of state or logic, declared outside the widget tree so it can be read, overridden and tested without one. `ref.watch` subscribes (rebuild on change); `ref.read` takes a one-off value, which is what you want inside a callback.

```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  // ✅ watch in build: the widget rebuilds when the invoice changes
  final invoice = ref.watch(invoiceProvider(id));

  return ElevatedButton(
    // ✅ read in a callback: call the API once, then refresh the invoice
    onPressed: () async {
      await ref.read(dioProvider).post('/invoices/$id/pay');
      ref.invalidate(invoiceProvider(id));
    },
    // ❌ ref.watch here would subscribe from inside a callback, which doesn't do what you expect
    child: const Text('Mark as paid'),
  );
}
```

* * *

## Folder structure

Feature-first:

```
lib/
├─ features/
│  └─ invoices/
│     ├─ data/          → API calls (Dio), local storage
│     ├─ domain/        → models, business rules
│     ├─ providers/     → Riverpod providers for this feature
│     └─ presentation/  → screens and widgets for this feature
├─ shared/              → reusable widgets, extensions, utils
├─ core/                → dio client, router, theme, constants
└─ main.dart
```

Same reasoning as the frontend page: group by feature, not by technical layer. Deleting a feature should be deleting a folder.

* * *

## Split widgets into widgets, not methods

The most common structural mistake in a Flutter codebase. A 300-line `build()` gets broken up — but into private *methods*, which fixes the reading problem and not the performance one.

```dart
// ❌ Looks tidy, but _buildHeader() has no identity of its own —
//    it rebuilds whenever anything in the parent rebuilds
class InvoiceScreen extends StatelessWidget {
  Widget _buildHeader() => const Padding(/* ... */);
  Widget _buildTotals() => Column(/* ... */);

  @override
  Widget build(BuildContext context) =>
      Column(children: [_buildHeader(), _buildTotals()]);
}

// ✅ A real widget class can be const, and rebuilds independently
class InvoiceScreen extends StatelessWidget {
  const InvoiceScreen({super.key});

  @override
  Widget build(BuildContext context) => const Column(
        children: [_InvoiceHeader(), _InvoiceTotals()],
      );
}

class _InvoiceHeader extends StatelessWidget {
  const _InvoiceHeader();
  @override
  Widget build(BuildContext context) => const Padding(/* ... */);
}
```

The underscore keeps the widget private to the file, so this costs nothing in API surface.

* * *

## Long lists use ListView builder

Same rule as `FlatList` on the [React Native side](Mobile-React-Native.md#long-lists-use-flatlist), same reason.

```dart
// ❌ Builds all 300 rows up front
ListView(children: invoices.map((i) => InvoiceCard(invoice: i)).toList())

// ✅ Builds what's visible, recycles the rest
ListView.builder(
  itemCount: invoices.length,
  itemBuilder: (context, index) => InvoiceCard(invoice: invoices[index]),
)
```

Plain `ListView` with a `children:` list is fine for a short, fixed set — a settings screen. Anything driven by an API response uses `.builder`.

* * *

## Keep logic out of widgets

A widget's job is to render state and forward events. Fetching, calculating and validating belong in a provider or a service — which is also the only way any of it becomes testable without pumping a widget tree.

| Belongs in | What |
| --- | --- |
| **Widget** | Layout, styling, wiring a callback to a provider method |
| **Provider (Riverpod)** | State, async loading, calling into the data layer |
| **`data/`** | The actual Dio call, the storage read/write |
| **`domain/`** | Models and business rules — pure Dart, no Flutter import |

> ⚠️ If a file in `domain/` needs `import 'package:flutter/material.dart'`, something has leaked in the wrong direction.

* * *

## Theme values

Colours, text styles and component defaults are defined once in `ThemeData` (see [Tech Stack → UI Framework](../Tech-Stack/Mobile-Stack-Alternative.md#ui-framework)). Widgets read them from the theme, so dark mode and a client rebrand only need a change in one place.

```dart
// ✅ follows the theme, including dark mode
final colors = Theme.of(context).colorScheme;
Container(
  color: colors.primary,
  child: Text('Paid', style: TextStyle(color: colors.onPrimary)),
)

// ❌ ignores dark mode and every future rebrand
Container(
  color: const Color(0xFF1D4ED8),
  child: const Text('Paid', style: TextStyle(color: Colors.white)),
)
```

* * *

## Calling the API

There is one configured `Dio` instance in `core/`, exposed through a provider. It holds the base URL, timeouts, and the interceptors that attach the token and handle a 401. Feature code always reads it from the provider. How it's set up is in [Tech Stack → HTTP Client](../Tech-Stack/Mobile-Stack-Alternative.md#http-client).

```dart
// ✅ features/invoices/providers/invoice_provider.dart: token and 401 handling already done
@riverpod
Future<Invoice> invoice(Ref ref, String id) async {
  final dio = ref.watch(dioProvider);
  final response = await dio.get('/invoices/$id');
  return Invoice.fromJson(response.data);
}

// ❌ a new Dio in feature code: no token, no timeout, hard-coded URL
final response = await Dio().get('http://localhost:3000/invoices/$id');
```

* * *

## Storing tokens and data

Pick the storage by **how sensitive the data is**, same rule as React Native.

| Data | Store | Why |
| --- | --- | --- |
| Tokens and credentials | **`flutter_secure_storage`** | Encrypted by the OS (Keychain / Keystore) |
| Cached data, preferences | **`hive_ce`** | Fast, but unencrypted |

```dart
// ✅ token: encrypted
await secureStorage.write(key: 'token', value: token);

// ✅ preference: not sensitive
await prefsBox.put('onboardingDone', true);

// ❌ token in Hive or shared_preferences: both are plain, unencrypted files
await prefsBox.put('token', token);
```

> ⚠️ Use the `hive_ce` package, not `hive`. The original package is unmaintained.

* * *

## Null safety

Dart's null safety is the equivalent of `strict: true`, and it's undermined the same two ways.

```dart
// ❌ "I know it's there" — until it isn't, and it's a production crash
final name = user!.name;

// ❌ late on something that genuinely might not be set yet
late final Invoice invoice;

// ✅ Handle the absence
final name = user?.name ?? 'Unknown';

if (user == null) return const SignInPrompt();
final name = user.name; // promoted to non-null, no ! needed
```

`!` is acceptable where you've *just* checked, or where the framework guarantees it. It is not a way to make the analyzer stop complaining.

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| `const` constructors wherever the analyzer allows | Leave `prefer_const_constructors` warnings unfixed | [Const constructors](#const-constructors) |
| Extract sub-widgets into private widget classes | `Widget _buildSomething()` helper methods | [Splitting widgets](#split-widgets-into-widgets-not-methods) |
| `ListView.builder` for API-driven lists | `ListView(children: [...])` for a long list | [Long lists](#long-lists-use-listview-builder) |
| `ref.watch` in `build`, `ref.read` in callbacks | `ref.watch` inside a callback — it won't do what you expect | [Providers](#riverpod-providers) |
| `if (!context.mounted) return;` after an `await` | Use a `BuildContext` across an async gap | [Async gaps](#buildcontext-and-async-gaps) |
| Theme values via `Theme.of(context)` | Hardcoded `Color(0xFF...)` scattered through widgets | [Theme values](#theme-values) |
| One configured `Dio` instance in `core/` | `Dio()` constructed ad hoc in feature code | [Calling the API](#calling-the-api) |
| `flutter_secure_storage` for tokens | Tokens in Hive or `shared_preferences` — both unencrypted | [Storage](#storing-tokens-and-data) |
| `hive_ce` | `hive` — the original package is unmaintained | [Storage](#storing-tokens-and-data) |
| Handle `loading` / `error` / `data` explicitly with `.when` | Show a blank screen while something loads | [Build runs constantly](#build-runs-constantly) |
| Handle the null with `?.`, `??` or an early return | `!` to make the analyzer stop complaining | [Null safety](#null-safety) |
| `dart format` + `dart analyze` before commit | Argue about formatting — the formatter isn't configurable, that's the point | [Linting](#linting-and-formatting) |

* * *

## Linting and formatting

Both ship in the SDK, so there's no tool to choose — only a config to set.

```yaml
# analysis_options.yaml
include: package:flutter_lints/flutter.yaml

linter:
  rules:
    prefer_const_constructors: true
    prefer_const_literals_to_create_immutables: true
```

- `dart format .` rewrites layout; it isn't configurable, and that's deliberate — see [General Principles → Linting and formatting](General-Principles.md#linting-and-formatting).
- `flutter analyze` runs the lints from `analysis_options.yaml`. `flutter_lints` is the official baseline; add rules on top rather than replacing it.
- Pre-commit: a simple git hook running `dart format --output=none --set-exit-if-changed .` and `flutter analyze` is enough — Flutter's tooling doesn't need a staged-files runner like `lint-staged`.
- CI: the same two commands as a merge gate. `--set-exit-if-changed` makes `dart format` fail the build instead of silently passing on unformatted code.

* * *

## Testing

Flutter ships its test framework in the SDK, and widget tests are cheap enough that there's no good excuse to skip them — this is a stricter expectation than the React Native page sets, and that's deliberate.

- **Unit tests** for `domain/` logic — pure Dart, fast, no widget tree.
- **Widget tests** for shared components: pump the widget, assert on what's rendered. Override providers with `ProviderContainer` / `ProviderScope(overrides: ...)` to supply test doubles.
- **Integration tests** for one or two critical end-to-end flows, not every screen.
- Write them alongside shared components from the first sprint. Retrofitted tests get written to match the bugs, not the requirements.

```dart
// invoice_screen_test.dart — swap the real provider for a fake, then check what's on screen
testWidgets('shows the invoice title', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        invoiceProvider('42').overrideWith((ref) async => Invoice(id: '42', title: 'March consulting')),
      ],
      child: const MaterialApp(home: InvoiceScreen(id: '42')),
    ),
  );
  await tester.pumpAndSettle();

  expect(find.text('March consulting'), findsOneWidget);
});
```

* * *

## Further reading

- [Flutter — Performance best practices](https://docs.flutter.dev/perf/best-practices)
- [Effective Dart — Style, Documentation, Usage, Design](https://dart.dev/effective-dart)
- [Flutter — Architecture guide](https://docs.flutter.dev/app-architecture)
- [Riverpod — Essentials](https://riverpod.dev/docs/essentials/first_request)

---

[⬅ Back to Coding Standards](README.md)
