# Angular Signals (Part II - The Signals Best Practice)

> Angular Singals best practices Workshops to explore the code benefits of the new reactive path

## In short: Angular Signal best practices Pattern
1. Never mutate a signal value in place
2. Don't use effect() to derive/sync state: use computed()
3. Don't declare signals/computed inside methods or functions
4. Set a custom equal comparator for object/array signals when needed
5. Always register cleanup for effects with subscriptions/timers
6. Use untracked() to break read/write feedback loops
7. Always provide initialValue to toSignal()

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


1. Never mutate a signal value in place
Calling .push()/.splice()/property-assignment on the object a signal returns doesn't notify subscribers. Always .set() or .update() with a new reference.

2. Don't use effect() to derive/sync state: use computed()
If an effect's only job is to compute a value from other signals and store it via .set()/.update(), that's what computed() is for. Effects should be reserved for side effects (logging, DOM, storage), not state derivation — using them for derivation causes extra change-detection cycles and ordering bugs.

3. Don't declare signals/computed inside methods or functions
signal()/computed() calls should live as class fields (or module-level), created once. Declaring them inside a method means a new signal instance is created on every call — reactivity breaks silently.

4. Set a custom equal comparator for object/array signals when needed
By default signals use reference equality (===). If you routinely .set() a new object/array with the same logical content, you'll trigger downstream recomputation/re-renders needlessly — pass { equal: ... } where deep-equality matters.

5. Always register cleanup for effects with subscriptions/timers
An effect() that starts a subscription or timer without an onCleanup callback leaks on re-run or destroy.

6. Use untracked() to break read/write feedback loops
An effect that reads signal A and writes signal B, where B is also read elsewhere and can affect A, risks infinite/extra runs. Wrap reads that shouldn't be tracked in untracked().

7. Always provide initialValue to toSignal()
Without it, the signal is undefined until the source observable emits — a common source of template/type errors when converting Observables to Signals.

---

## Automaticly Enforce Signal best-practices

### Basic Audit greps / smoke tests
```bash
## Linux
grep -rn "().push\|().pop\|().splice" src/     # basic smoke test rule 1
grep -rn "effect(" src/ | wc -l                # basic smoke test 3 and 7
grep -rn "{{ [a-z]*(" src/ --include="*.html"  # basic smoke test 4
```

### Full Audit greps / smoke tests
```bash
## Linux
# 1. Signal mutated in place (push/splice/etc. or direct prop assignment on a signal read)
grep -rnE '\(\)\.(push|pop|shift|unshift|splice|sort|reverse|fill)\(' --include=*.ts src/app
grep -rnE '\(\)\.[A-Za-z_][A-Za-z0-9_]*\s*=[^=]' --include=*.ts src/app
grep -rnE '\(\)\[[^\]]*\]\s*=[^=]' --include=*.ts src/app

# 2. effect() that calls .set()/.update() — likely should be computed() instead
grep -rnE -A3 'effect\(' --include=*.ts src/app | grep -B3 -E '\.set\(|\.update\('

# 3. signal()/computed() declared inside a function/method instead of as a class field
grep -rnE '^\s*(const|let)\s+\w+\s*=\s*(signal|computed)\(' --include=*.ts src/app

# 4. signal<Array|Object> without a custom equal comparator — needs manual eyeballing
grep -rnE 'signal<[^>]*(\[\]|\{)' --include=*.ts src/app

# 5. effect() usage vs cleanup registration — big gap suggests missing cleanup somewhere
grep -rcE 'effect\(' --include=*.ts src/app | awk -F: '{s+=$2} END{print "effect() calls:", s}'
grep -rcE 'onCleanup' --include=*.ts src/app | awk -F: '{s+=$2} END{print "onCleanup uses:", s}'

# 6. effect() usage vs untracked() usage — zero untracked with multiple cross-writing effects is a smell
grep -rcE 'untracked\(' --include=*.ts src/app | awk -F: '{s+=$2} END{print "untracked() uses:", s}'

# 7. toSignal() without initialValue
grep -rnE -A3 'toSignal\(' --include=*.ts src/app | grep -B3 -v 'initialValue'
```

## THANKS!

### Workshop Recording Link
[Watch the Teams Meeting Recording - 2025/10/24](https://teams.microsoft.com/l/meetingrecap?driveId=b%21eE0f-puY2E2k9oszmlxnJUW0OIvoWIpKg9NjqSqLhFF4YTqeCMNgRo9dsBlMI30w&driveItemId=01UYUOEWGO3FDLNONBFRE3IVCJO2JF5EDW&sitePath=https%3A%2F%2Feque2online-my.sharepoint.com%2F%3Av%3A%2Fg%2Fpersonal%2Fleo_lanese_eque2_com%2FEc7ZRra5oSxJtFRJdpJekHYB4xvQL5LOIQ5XjD-iynmlHQ&fileUrl=https%3A%2F%2Feque2online-my.sharepoint.com%2F%3Av%3A%2Fg%2Fpersonal%2Fleo_lanese_eque2_com%2FEc7ZRra5oSxJtFRJdpJekHYB4xvQL5LOIQ5XjD-iynmlHQ&iCalUid=040000008200E00074C5B7101A82E00800000000BBEA13F14F3FDC01000000000000000010000000B6E71E2AB3CFFF49A985053F80FE66D1&threadId=19%3Ameeting_YTljZDc4MTgtNDAxZS00MDk1LWIzYTQtMDIwOThiZjMwNWVl%40thread.v2&organizerId=9948fa2f-c392-4610-9eb4-535f8348b501&tenantId=30cbef6a-55b1-4f4e-b86a-ab4f70fafa33&callId=dbb2aec7-2674-4ce9-aaf1-9de18edb3e5e&threadType=Meeting&meetingType=Scheduled&subType=RecapSharingLink_RecapChiclet)

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

