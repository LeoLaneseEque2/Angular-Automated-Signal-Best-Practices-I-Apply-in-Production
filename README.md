# Angular Automated Signal Best Practices I Apply in Production -Angular Workshop - Eque2 (©)

> Angular Signals best practices Eque2 Workshop to explore the code benefits of the new Angular reactive path

The idea is to catch Signal Anti-Patterns before they hit production. These are the rules I hold every signal-driven feature to. Each one comes down to the same idea: work with Angular's reactivity graph instead of around it. Get these right and signals do exactly what they promise, fine-grained updates, no wasted recomputation, predictable data flow.

## This article has two parts:
1. Signal Best practices: Description, rule, examples
2. Automated Signal Best Practices:What catches, Basic Audit greps, full script

---

## In short: Angular Signal best practices Pattern
1. Never mutate a signal value in place
2. Don't use effect() to derive/sync state: use computed()
3. Don't declare signals/computed inside methods or functions
4. Set a custom equal comparator for object/array signals when needed
5. Always register cleanup for effects with subscriptions/timers
6. Use untracked() to break read/write feedback loops
7. Always provide initialValue to toSignal()
8. Use resource() to fetch signal-dependent async data while automatically handling loading, error, and value states without manual wiring.

## Web implemented Signal Benefits:
- .subscription() -> toSignal() = for automatic cleanup
- Use computed() for derived pagination values
- remove manual ngOnDestroy subscription management
- Automatic reactivity: currentPageNum or perPage change
- Simplify initialisation logic
- Pure signals: no Observable interop (except minimal Promise conversion)
- Automatic reactivity: effect runs when signals change
- Automatic cleanup: effects are cleaned up automatically
- Simpler code: no RxJS operators or Observable chains
- Type-safe: all signals are strongly typed

---

## Explanation 

1. ⚠️ Treat Signal Values as Immutable
A signal only notifies its subscribers when you call .set() or .update(). Reaching into the current value and changing it directly, pushing into an array, splicing it, assigning a property on an object, never produces a new reference, so there's nothing for Angular to compare and nothing to notify. The rule I hold everyone to: every write to a signal produces a brand new value.

```typescript
export class MemberListStore {
  members = signal<GymMember[]>([]);

  addMember(member: GymMember) {
    this.members.update(members => [...members, member]);
  }

  removeMember(id: string) {
    this.members.update(members => members.filter(m => m.id !== id));
  }
}
```

2. ⚠️ Keep computed() Pure
computed() is meant to be a pure, synchronous derivation, nothing else. Angular can re-evaluate it more than once before its value is actually read, and it can re-run any time an upstream signal changes, whether or not anything is reading the result yet. I keep anything that isn't a straight derivation, HTTP calls, writes to other signals, logging, localStorage access, out of it entirely, because inside a computed() that work runs at a frequency I don't control.

```typescript
totalPrice = computed(() => {
  const tier = this.selectedTier();
  const members = this.memberCount();
  return calculateElasticPrice(tier, members); // pure function, no I/O
});

// anything side-effecting, like analytics, lives in an effect and stays minimal
constructor() {
  effect(() => {
    const price = this.totalPrice();
    untracked(() => this.analytics.track('price_calculated', { price }));
  });
}
```

3. ⚠️ Derive State, Don't Sync It With effect()
Copying one signal's value into another signal inside an effect() builds a manual, imperative sync path for something computed() already handles for free. It's an extra change detection tick, it's harder to trace, and it can loop if you're not deliberate about what the effect reads versus writes. When I need a resettable or overridable derived value, which comes up constantly in stepper UIs, I reach for linkedSignal() instead of an effect.

```typescript
// derive it directly
canContinue = computed(() => this.planSelected() && this.paymentDetailsValid());

// when I need an overridable derived value, linkedSignal is the right tool
currentStep = signal(0);
// resets to a computed default whenever the source changes, but stays writable
draftStepLabel = linkedSignal(() => this.stepDefinitions()[this.currentStep()].label);
```

4. ⚠️ Push Derived Values Through computed(), Not Template Functions
Calling a method in a template, {{ getTotal() }}, re-executes it on every change detection run regardless of whether its inputs changed, because Angular has no way to know the function is pure or what it depends on. Zoneless cuts down how often that happens overall, but it still re-runs on every relevant signal-driven pass instead of only when a real dependency changes. I always route derived template values through computed(), which only recalculates when a tracked dependency actually changes and caches the result otherwise.

```typescript
monthlyRecurringRevenue = computed(() =>
  this.memberships()
    .filter(m => m.status === 'active')
    .reduce((sum, m) => sum + m.monthlyFee, 0)
);
...
<p>MRR: {{ monthlyRecurringRevenue() | currency:'GBP' }}</p>
```

5. ⚠️ Keep Signals Granular, Never One Signal Holding Everything
Before I ship a store, I run it through the same test: "Open your biggest signal and ask: does everything in here change together? If not, split it."
A single large signal holding unrelated state means any update to any field invalidates every computed() and effect() reading the whole object, even the ones that only care about a field that didn't change. Independently changing signals mean each consumer only re-runs for the slice it actually depends on.   

```typescript
memberCount = signal(42);
selectedTier = signal<'elastic-lite' | 'elastic-pro' | 'elastic-max'>('elastic-pro');
themeMode = signal<'dark' | 'light'>('dark');
searchQuery = signal('');
isSidebarOpen = signal(true);

// pricing only recomputes when the 2 signals it actually needs change
totalPrice = computed(() => calculateElasticPrice(this.selectedTier(), this.memberCount()));
```

6. ⚠️ Guard Derived Objects Against Reference Churn
Signals and computed() only skip notifying downstream consumers when the new value is equal to the old one by reference. If a computed() returns a new object or array literal on every run, even one structurally identical to the last result, every consumer treats it as changed: OnPush components re-render, effects re-fire, @Input() bindings churn. I check every computed() that returns an object or array for exactly this, since it's the one place fine-grained reactivity can quietly turn into the opposite of what it promises.

```typescript
// option A: supply a custom equality function
memberDisplay = computed(
  () => ({ name: this.member().name, status: this.member().status }),
  { equal: (a, b) => a.name === b.name && a.status === b.status }
);

// option B: split into primitive signals instead, usually simpler
memberName = computed(() => this.member().name);
memberStatus = computed(() => this.member().status);
```

7. ⚠️ Reserve effect() for True Side Effects
I keep effect() scoped to work that genuinely lives outside Angular's reactivity graph: writing to localStorage, calling a non-Angular library, manual DOM work, logging, analytics. Everything else already has a declarative mechanism, template bindings, computed(), @if/@for, that's optimized, tracked, and cleaned up automatically. Using effect() for anything those can do means rebuilding what Angular's binding system already does, on an untracked path that's harder to reason about and can fire more often than expected, including once immediately on creation.

```typescript
@for (step of steps(); track step.id; let i = $index) {
  <div class="step" [class.step--active]="i === currentStep()">
    {{ step.label }}
  </div>
}
...
constructor() {
  effect(() => {
    const draft = this.onboardingForm.getRawValue();
    untracked(() => localStorage.setItem('onboarding-draft', JSON.stringify(draft)));
  });
}
```

8. ⚠️ Use resource() for Async Data Tied to a Signal
Practices 2 and 3 rule out doing async work inside computed() and rule out using effect() to hand-roll state syncing. That leaves an obvious question: what's the right way to fetch data that depends on a signal? For that I reach for resource(), which ties an async load directly to a signal input and gives me loading, error, and value state for free, without wiring up a data signal, a loading signal, and an error signal by hand and keeping all three in sync myself.

`resource()` reloads automatically whenever the signals read inside params change, and it cancels an in-flight request if a new one supersedes it, through the abortSignal it passes to the loader. Loading a location's member roster looks like this:

```typescript
selectedLocationId = signal<string>('loc-001');

memberRoster = resource({
  params: () => ({ locationId: this.selectedLocationId() }),
  loader: ({ params, abortSignal }) =>
    this.membersApi.fetchRoster(params.locationId, { signal: abortSignal })
});

...

@if (memberRoster.isLoading()) {
  <app-spinner />
} @else if (memberRoster.error(); as err) {
  <app-error-banner [error]="err" />
} @else {
  <app-member-list [members]="memberRoster.value() ?? []" />
}
```
One detail I keep in mind: only signals read inside params are tracked. A signal read inside loader itself won't trigger a reload, since Angular can't track dependencies across an await. Anything the loader needs has to come in through params.


| # | Practice | Why it matters | How I apply it |
|---:|----------|----------------|----------------|
| 1 | Treat signal values as immutable | Direct mutation never triggers notification | `.update()` / `.set()` with new references |
| 2 | Keep `computed()` pure | Side effects inside it run at an unpredictable frequency | Pure derivations only. Side effects go in `effect()` |
| 3 | Derive state, don't sync it | `effect()`-driven sync adds ticks and can loop | Use `computed()`. Use `linkedSignal()` for overridable derived state |
| 4 | Push derived values through `computed()` | Template function calls re-run on every change detection pass | Replace template functions with `computed()` |
| 5 | Keep signals granular | One large signal invalidates everything on any change | Split into independently changing signals |
| 6 | Guard against reference churn | Fresh object/array literals always read as "changed" | Use a custom `equal` comparator, or split into primitives |
| 7 | Reserve `effect()` for true side effects | Declarative bindings already cover the rest | Prefer template bindings and `computed()`. Use `effect()` only for real externalities |
| 8 | Use `resource()` for signal-driven async data | Manual data/loading/error signals are easy to get out of sync | Use `resource()` with `params` and `loader`, driven by a signal |

---

## 2. Automated Signal Best Practices

### Basic Audit greps / smoke tests
```bash
## Linux
grep -rn "().push\|().pop\|().splice" src/     # basic smoke test rule 1
grep -rn "effect(" src/ | wc -l                # basic smoke test 3 and 7
grep -rn "{{ [a-z]*(" src/ --include="*.html"  # basic smoke test 4
```

### What it catches:
- 🚫 Hard Fails: In-place mutations (.push(), .splice())
- ⚠️ Warnings: effect() deriving state (use computed())
- ⚠️ Warnings: Async fetching in effect() (use resource())
- 📊 Stats: effect() vs onCleanup() ratios (leak detection)

### Full Audit greps / smoke tests
```bash
# 1. In-place mutations
grep -rn "\(\)\.push\|\(\)\.splice" src/         

# 2. effect() deriving state
grep -A5 "effect(" src/ | grep "\.set\("          

# 3. Signals inside functions
grep -rn "const .* = signal(" src/          

# 4. Array signals (needs review)      
grep -rn "signal<.*\[\]>" src/     

# 5-6. Cleanup/untracked stats               
grep -c "effect(" src/ | awk '{s+=$1}...'   

# 7. Missing initial values     
grep -A3 "toSignal(" src/ | grep -v "initialValue"

# 8. Async in effect()`
grep -A5 "effect(" src/ | grep "\.subscribe\|http\." 
```

### 🔗 Grab the full script here:
https://gist.github.com/leolanese/8443691d4973634ca2188ab42c9ce42a

## THANKS!

## 📚 Previous Workshop

Before diving into these best practices, you may find it useful to review the fundamentals of Angular Signals.

**🔗 Angular Signals Workshop**  
https://github.com/LeoLaneseEque2/Angular-Signals-Workshop



MIT license
====================
Or same license apply for 3rd party libraries I'm using if apply.

---

## 📬 Reach me

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/leolanese/)
[![Dev.to](https://img.shields.io/badge/dev-000000?style=for-the-badge&logo=black&logoColor=white)](http://www.dev.to/leolanese)
[![Twitter](https://img.shields.io/badge/Twitter-1DA1F2?style=for-the-badge&logo=twitter&logoColor=white)](http://twitter.com/LeoLanese)
[![Blog](https://img.shields.io/badge/blog-ededed?style=for-the-badge)](http://www.leolanese.com/blog)
[![Email](https://img.shields.io/badge/email-Developer%40leolanese.com-informational?style=for-the-badge)](mailto:engineer@leolanese.com)

