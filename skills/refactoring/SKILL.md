# Refactoring Rules

## 1. Refactoring Principles

### Core Definition

- Refactoring is changing the internal structure of code
  without altering its observable behavior
- The goal is to make code easier to understand,
  modify, and extend

### Fundamental Rules

| Rule                   | Description                                               |
| ---------------------- | --------------------------------------------------------- |
| Behavior preservation  | External behavior must remain identical before and after  |
| Small steps            | Change one thing at a time, verify after each step        |
| Test accompaniment     | Every refactoring step must be validated by passing tests |
| Separation of concerns | Never mix refactoring with feature addition or bug fixes  |
| Reversibility          | Each step should be easily reversible if something breaks |

### Refactoring Workflow

```text
1. Ensure tests pass (green)
2. Apply one small refactoring
3. Run tests again (must stay green)
4. Commit
5. Repeat
```

- If tests fail after a refactoring step,
  revert immediately rather than debugging forward
- Compile-and-run after every change — do not batch multiple refactorings

---

## 2. Code Smell Catalog

### Method-Level Smells

| Smell               | Symptom                                                          | Typical Fix                   |
| ------------------- | ---------------------------------------------------------------- | ----------------------------- |
| Long Method         | Method exceeds 20-30 lines or has multiple levels of abstraction | Extract Method                |
| Long Parameter List | Method takes 4+ parameters                                       | Introduce Parameter Object    |
| Primitive Obsession | Using primitives instead of small domain objects                 | Replace Primitive with Object |

### Class-Level Smells

| Smell           | Symptom                                                 | Typical Fix                         |
| --------------- | ------------------------------------------------------- | ----------------------------------- |
| Large Class     | Class has too many fields, methods, or responsibilities | Extract Class, Extract Superclass   |
| Data Class      | Class only holds data with no meaningful behavior       | Move behavior into the class        |
| Refused Bequest | Subclass ignores most inherited methods/properties      | Replace Inheritance with Delegation |

### Coupling Smells

| Smell                  | Symptom                                                 | Typical Fix                      |
| ---------------------- | ------------------------------------------------------- | -------------------------------- |
| Feature Envy           | Method accesses another object's data more than its own | Move Method to the envied class  |
| Inappropriate Intimacy | Two classes access each other's internals excessively   | Move Method/Field, Extract Class |
| Message Chains         | `a.getB().getC().getD().doSomething()`                  | Hide Delegate, Extract Method    |
| Middle Man             | Class delegates almost everything to another class      | Remove Middle Man, Inline Class  |

### Change-Pattern Smells

| Smell             | Symptom                                              | Typical Fix                               |
| ----------------- | ---------------------------------------------------- | ----------------------------------------- |
| Divergent Change  | One class is modified for multiple unrelated reasons | Extract Class (split by responsibility)   |
| Shotgun Surgery   | One change requires edits across many classes        | Move Method/Field to consolidate          |
| Data Clumps       | Same group of fields appears in multiple places      | Introduce Parameter Object, Extract Class |
| Switch Statements | Same switch/when structure duplicated across methods | Replace Conditional with Polymorphism     |

### Detection Heuristics

- If a method needs a comment to explain what it does — it is too complex
- If you scroll to read a method — it is too long
- If changing one feature touches 5+ files — suspect Shotgun Surgery
- If a class changes for 3+ different reasons — suspect Divergent Change

---

## 3. Basic Refactoring Techniques

### Extract Method

Move a code fragment into a named method whose name explains its purpose.

```kotlin
// Before
fun printOwing(invoice: Invoice) {
    var outstanding = 0.0
    for (order in invoice.orders) {
        outstanding += order.amount
    }
    println("name: ${invoice.customer}")
    println("amount: $outstanding")
}

// After
fun printOwing(invoice: Invoice) {
    val outstanding = calculateOutstanding(invoice)
    printDetails(invoice, outstanding)
}

private fun calculateOutstanding(invoice: Invoice): Double =
    invoice.orders.sumOf { it.amount }

private fun printDetails(invoice: Invoice, outstanding: Double) {
    println("name: ${invoice.customer}")
    println("amount: $outstanding")
}
```

### Inline Method

Replace a method call with the method body
when the body is as clear as the name.

```kotlin
// Before
fun getRating(): Int = if (moreThanFiveLateDeliveries()) 2 else 1
private fun moreThanFiveLateDeliveries(): Boolean = lateDeliveries > 5

// After
fun getRating(): Int = if (lateDeliveries > 5) 2 else 1
```

### Extract Variable

Replace a complex expression with a named variable
that explains its intent.

```kotlin
// Before
fun price(order: Order): Double =
    order.quantity * order.itemPrice -
        maxOf(0.0, order.quantity - 500) * order.itemPrice * 0.05 +
        minOf(order.quantity * order.itemPrice * 0.1, 100.0)

// After
fun price(order: Order): Double {
    val basePrice = order.quantity * order.itemPrice
    val quantityDiscount = maxOf(0.0, order.quantity - 500) * order.itemPrice * 0.05
    val shipping = minOf(basePrice * 0.1, 100.0)
    return basePrice - quantityDiscount + shipping
}
```

### Inline Variable

Remove a variable that adds no explanation
beyond the expression itself.

```kotlin
// Before
val basePrice = order.basePrice()
return basePrice > 1000

// After
return order.basePrice() > 1000
```

### Rename

Change a name to clearly communicate its purpose.

```kotlin
// Before
fun calc(a: Double, b: Double): Double = a * b * TAX_RATE

// After
fun calculateTaxedPrice(basePrice: Double, quantity: Double): Double =
    basePrice * quantity * TAX_RATE
```

### Move Method / Move Field

Relocate a method or field to the class
that uses it most.

```kotlin
// Before: Account calculates overdraft charge using AccountType data
class Account(val type: AccountType, val daysOverdrawn: Int) {
    fun overdraftCharge(): Double =
        if (type.isPremium) (daysOverdrawn - 7) * 0.85 else daysOverdrawn * 1.75
}

// After: AccountType owns the logic about its own charging rules
class AccountType(val isPremium: Boolean) {
    fun overdraftCharge(daysOverdrawn: Int): Double =
        if (isPremium) (daysOverdrawn - 7) * 0.85 else daysOverdrawn * 1.75
}
```

### Encapsulate Field

Replace direct field access with getter/setter
to control access and enable future changes.

```kotlin
// Before
class Person(var name: String)

// After
class Person(name: String) {
    var name: String = name
        private set

    fun rename(newName: String) {
        require(newName.isNotBlank()) { "Name must not be blank" }
        name = newName
    }
}
```

---

## 4. Simplifying Conditionals

### Replace Nested Conditionals with Guard Clauses

```kotlin
// Before: nested conditionals obscure the main path
fun payAmount(employee: Employee): Double {
    if (employee.isSeparated) {
        return separatedAmount()
    } else {
        if (employee.isRetired) {
            return retiredAmount()
        } else {
            return normalPayAmount(employee)
        }
    }
}

// After: guard clauses for special cases, main path at the end
fun payAmount(employee: Employee): Double {
    if (employee.isSeparated) return separatedAmount()
    if (employee.isRetired) return retiredAmount()
    return normalPayAmount(employee)
}
```

### Decompose Conditional

Extract complex condition logic into named functions.

```kotlin
// Before
fun charge(date: LocalDate, quantity: Int): Double =
    if (date.isBefore(SUMMER_START) || date.isAfter(SUMMER_END))
        quantity * winterRate + winterServiceCharge
    else
        quantity * summerRate

// After
fun charge(date: LocalDate, quantity: Int): Double =
    if (isWinter(date)) winterCharge(quantity)
    else summerCharge(quantity)

private fun isWinter(date: LocalDate): Boolean =
    date.isBefore(SUMMER_START) || date.isAfter(SUMMER_END)

private fun winterCharge(quantity: Int): Double =
    quantity * winterRate + winterServiceCharge

private fun summerCharge(quantity: Int): Double =
    quantity * summerRate
```

### Replace Conditional with Polymorphism

When the same `when`/`switch` structure appears in multiple places,
use a class hierarchy instead.

```kotlin
// Before: when on type repeated in multiple methods
fun calculatePay(employee: Employee): Double = when (employee.type) {
    EmployeeType.ENGINEER -> employee.monthlySalary
    EmployeeType.SALESPERSON -> employee.monthlySalary + employee.commission
    EmployeeType.MANAGER -> employee.monthlySalary + employee.bonus
}

// After: polymorphic behavior
sealed class Employee(val monthlySalary: Double) {
    abstract fun calculatePay(): Double
}

class Engineer(monthlySalary: Double) : Employee(monthlySalary) {
    override fun calculatePay(): Double = monthlySalary
}

class Salesperson(
    monthlySalary: Double,
    val commission: Double
) : Employee(monthlySalary) {
    override fun calculatePay(): Double = monthlySalary + commission
}

class Manager(
    monthlySalary: Double,
    val bonus: Double
) : Employee(monthlySalary) {
    override fun calculatePay(): Double = monthlySalary + bonus
}
```

### Consolidate Conditional Expression

Combine multiple conditions that produce the same result.

```kotlin
// Before
fun disabilityAmount(employee: Employee): Double {
    if (employee.seniority < 2) return 0.0
    if (employee.monthsDisabled > 12) return 0.0
    if (employee.isPartTime) return 0.0
    return computeDisabilityAmount(employee)
}

// After
fun disabilityAmount(employee: Employee): Double {
    if (isNotEligibleForDisability(employee)) return 0.0
    return computeDisabilityAmount(employee)
}

private fun isNotEligibleForDisability(employee: Employee): Boolean =
    employee.seniority < 2 ||
        employee.monthsDisabled > 12 ||
        employee.isPartTime
```

---

## 5. Organizing Data

### Replace Primitive with Object

Wrap a primitive that carries domain meaning
into a dedicated type.

```kotlin
// Before: raw String used for phone number everywhere
class Customer(
    val name: String,
    val phone: String  // format? validation? unknown
)

// After: dedicated value class with validation
@JvmInline
value class PhoneNumber(val value: String) {
    init {
        require(value.matches(Regex("^\\d{2,3}-\\d{3,4}-\\d{4}$"))) {
            "Invalid phone number format: $value"
        }
    }
}

class Customer(
    val name: String,
    val phone: PhoneNumber
)
```

### Introduce Parameter Object

Replace a recurring group of parameters
with a dedicated object.

```kotlin
// Before: same three parameters appear in multiple methods
fun amountInvoiced(startDate: LocalDate, endDate: LocalDate, currency: Currency): Money = ...
fun amountReceived(startDate: LocalDate, endDate: LocalDate, currency: Currency): Money = ...
fun amountOverdue(startDate: LocalDate, endDate: LocalDate, currency: Currency): Money = ...

// After: group into a parameter object
data class DateRange(val start: LocalDate, val end: LocalDate) {
    init { require(!start.isAfter(end)) { "Start must not be after end" } }
    fun contains(date: LocalDate): Boolean = !date.isBefore(start) && !date.isAfter(end)
}

fun amountInvoiced(range: DateRange, currency: Currency): Money = ...
fun amountReceived(range: DateRange, currency: Currency): Money = ...
fun amountOverdue(range: DateRange, currency: Currency): Money = ...
```

### Preserve Whole Object

Pass the whole object instead of extracting multiple values from it.

```kotlin
// Before: extracting values just to pass them
val low = daysTempRange.low
val high = daysTempRange.high
val isWithinRange = plan.withinRange(low, high)

// After: pass the object itself
val isWithinRange = plan.withinRange(daysTempRange)
```

### Replace Type Code with Sealed Class

Replace type code constants with a type-safe hierarchy.

```kotlin
// Before: magic constants
class Employee(val type: Int) {
    companion object {
        const val ENGINEER = 0
        const val SALESPERSON = 1
        const val MANAGER = 2
    }
}

// After: sealed class hierarchy
sealed class EmployeeType {
    data object Engineer : EmployeeType()
    data object Salesperson : EmployeeType()
    data object Manager : EmployeeType()
}
```

---

## 6. API Refactoring

### Rename Method

Change method name to clearly describe what it does.

```kotlin
// Before
fun inv(c: Customer): List<Invoice> = ...

// After
fun findInvoicesByCustomer(customer: Customer): List<Invoice> = ...
```

### Add / Remove Parameter

Adjust the parameter list to match actual usage.

```kotlin
// Add: method needs additional context
// Before
fun createOrder(items: List<Item>): Order = ...
// After
fun createOrder(items: List<Item>, customer: Customer): Order = ...

// Remove: parameter is no longer used
// Before
fun applyDiscount(order: Order, season: String): Order = ...  // season unused
// After
fun applyDiscount(order: Order): Order = ...
```

### Replace Parameter with Method Call

Remove a parameter whose value can be obtained
from an existing object.

```kotlin
// Before: caller computes and passes derived value
val basePrice = quantity * itemPrice
val discount = getDiscount(basePrice, quantity, itemPrice)

// After: method computes what it needs internally
fun getDiscount(quantity: Int, itemPrice: Double): Double {
    val basePrice = quantity * itemPrice
    return if (basePrice > 1000) 0.05 else 0.0
}
```

### Separate Query from Modifier

A function should either return a value (query)
or change state (command), not both.

```kotlin
// Before: mixed query and modifier
fun getTotalAndApplyDiscount(order: Order): Double {
    order.applyDiscount(0.1)  // side effect
    return order.total         // query
}

// After: separated
fun applyDiscount(order: Order, rate: Double) {
    order.applyDiscount(rate)
}

fun getTotal(order: Order): Double = order.total
```

---

## 7. When to Refactor

### Timing Guidelines

| Trigger                   | Description                                                       |
| ------------------------- | ----------------------------------------------------------------- |
| Rule of Three             | Tolerate duplication once; on the third occurrence, refactor      |
| Before adding a feature   | Clean the area you are about to modify to make the change easier  |
| During code review        | Reviewer identifies structural issues — refactor before merging   |
| While fixing a bug        | If the bug area is hard to understand, refactor first for clarity |
| After learning new intent | When you understand the domain better, rename and restructure     |

### When NOT to Refactor

| Situation                            | Reason                                         |
| ------------------------------------ | ---------------------------------------------- |
| Code is stable and never read        | No benefit; risk of introducing bugs           |
| Rewrite is more practical            | Refactoring assumes working code worth keeping |
| Under extreme time pressure          | Partial refactoring can leave code worse       |
| No test coverage                     | Cannot verify behavior preservation            |
| Across API boundaries with consumers | Breaking changes require coordinated migration |

### Refactoring Scope Control

- Set a time limit (e.g., 30 minutes) for opportunistic refactoring
- If the refactoring grows beyond the limit, create a separate task/ticket
- Never let refactoring scope creep into feature work

---

## 8. Refactoring Safety Net

### Test Coverage

- Refactoring without tests is guessing, not refactoring
- Ensure the area being refactored has adequate test coverage before starting
- If tests are missing, write characterization tests first
  (tests that document current behavior, even if imperfect)

### IDE Automation

| Refactoring             | IDE Support | Trust Level |
| ----------------------- | ----------- | ----------- |
| Rename                  | Excellent   | High        |
| Extract Method          | Good        | High        |
| Inline Method/Variable  | Good        | High        |
| Move Class/Method       | Good        | Medium      |
| Change Signature        | Good        | Medium      |
| Extract Interface/Class | Moderate    | Medium      |
| Complex restructuring   | Limited     | Low         |

- Prefer IDE automated refactoring over manual editing whenever possible
- IDE refactoring is less error-prone because it handles all references
- Always run tests after IDE-automated refactoring — trust but verify

### Commit Strategy

- Commit after each successful refactoring step
- Keep refactoring commits separate from feature/bugfix commits
- Use conventional commit type: `refactor(scope): description`
- Small commits enable precise `git bisect` if something breaks later

### Characterization Tests

```kotlin
// When refactoring legacy code without tests,
// write tests that capture current behavior first
describe("LegacyPriceCalculator") {
    context("현재 동작을 기록하는 특성 테스트") {
        it("기본 가격에 세금을 포함하여 반환한다") {
            val calculator = LegacyPriceCalculator()
            // Capture current behavior — even if the value seems wrong
            calculator.calculate(100.0) shouldBe 110.0
        }
    }
}
```

---

## 9. Anti-Patterns

### Big Bang Refactoring

- Attempting to refactor an entire module or system at once
- Takes days or weeks, impossible to verify behavior preservation
- High risk of introducing bugs and merge conflicts
- Instead: refactor incrementally, one small change at a time

### Refactoring Without Tests

- Changing code structure without a way to verify correctness
- Bugs introduced during refactoring go undetected
- Instead: write characterization tests before refactoring untested code

### Mixing Refactoring with Feature Addition

- Adding new behavior while restructuring existing code
- If something breaks, impossible to tell which change caused it
- Instead: refactor first (commit), then add the feature (separate commit)

### Speculative Generalization

- Adding abstraction layers, parameters, or extension points
  "because we might need them someday"
- Increases complexity without immediate benefit
- Instead: keep it simple; refactor to generalize when the need arises

### Incomplete Refactoring

- Starting a refactoring but stopping halfway
  (e.g., extracting a class but not moving all relevant methods)
- Leaves code in a worse state — responsibility split across two places
- Instead: finish the refactoring or revert entirely

### Refactoring for Style Preference

- Rewriting code that works correctly and is readable
  just because you prefer a different style
- Introduces unnecessary churn and merge conflicts
- Instead: adopt team conventions and only refactor for measurable improvement

---

## 10. Related Rules

| Rule File         | Relevance                                                              |
| ----------------- | ---------------------------------------------------------------------- |
| `code-quality.md` | Quality pillars that refactoring aims to achieve                       |
| `code-review.md`  | Code review is a key trigger for identifying refactoring opportunities |
| `testing-unit.md`     | Tests provide the safety net for refactoring                           |
| `git-workflow.md` | Commit conventions for refactoring changes                             |

---

## 11. Additional Learning Resources

- Martin Fowler, *Refactoring: Improving the Design of Existing Code* (2nd Edition)
- Joshua Kerievsky, *Refactoring to Patterns*
- Michael Feathers, *Working Effectively with Legacy Code*
- Sandro Mancuso, *The Software Craftsman*
- JetBrains IntelliJ IDEA Refactoring Documentation
