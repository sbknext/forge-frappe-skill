---
title: Frappe Ref — Frappe Errors: Controllers: Anti Patterns (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'errors', 'controllers', 'reference', 'anti-patterns']
source: Impertio
---

# Controller Anti-Patterns — Error Prevention

Each anti-pattern shows the mistake, why it fails, and the correct approach.

---

## 1. Calling self.save() in Hooks

```python
# ❌ WRONG — Infinite recursion
def validate(self):
    self.calculate()
    self.save()  # Triggers validate again

def on_update(self):
    self.status = "Done"
    self.save()  # Triggers on_update again

# ✅ CORRECT
def validate(self):
    self.calculate()
    # No save — framework handles it

def on_update(self):
    self.db_set("status", "Done")  # Direct DB, no trigger
```
**Why:** `save()` triggers the same hook, causing infinite recursion.

---

## 2. Missing super() in Overrides

```python
# ❌ WRONG — All parent logic bypassed
class CustomInvoice(SalesInvoice):
    def validate(self):
        self.my_check()  # ERPNext validations skipped!

# ✅ CORRECT
class CustomInvoice(SalesInvoice):
    def validate(self):
        super().validate()  # Run parent first
        self.my_check()
```
**Why:** Without `super()`, critical framework and parent app validations are silently skipped.

---

## 3. frappe.db.commit() in Controllers

```python
# ❌ WRONG — Breaks transaction management
def validate(self):
    frappe.db.set_value("Counter", "main", "count", 1)
    frappe.db.commit()  # Committed even if save fails later!

# ✅ CORRECT
def validate(self):
    frappe.db.set_value("Counter", "main", "count", 1)
    # Framework commits everything together, or rolls back together
```
**Why:** Manual commits break Frappe's request-level transaction, causing partial saves.

---

## 4. Changes in on_update Without db_set()

```python
# ❌ WRONG — Changes lost
def on_update(self):
    self.sync_status = "Synced"  # NOT saved!

# ✅ CORRECT
def on_update(self):
    self.db_set("sync_status", "Synced")
```
**Why:** After `on_update`, the document is already committed. Changes to `self` are not persisted.

---

## 5. Validation in on_submit (Too Late)

```python
# ❌ WRONG — Document already submitted!
def on_submit(self):
    if self.grand_total > self.credit_limit:
        frappe.throw("Credit limit exceeded")  # docstatus already 1!

# ✅ CORRECT — Validate in before_submit
def before_submit(self):
    if self.grand_total > self.credit_limit:
        frappe.throw("Credit limit exceeded")  # Clean abort, stays Draft
```
**Why:** In `on_submit`, docstatus is already 1. Throwing creates an inconsistent state.

---

## 6. Swallowing Errors Silently

```python
# ❌ WRONG — Debugging impossible
def validate(self):
    try:
        self.check_stock()
    except Exception:
        pass  # What went wrong? Nobody knows.

# ✅ CORRECT
def validate(self):
    try:
        self.check_stock()
    except frappe.ValidationError:
        raise  # Re-raise validation errors
    except Exception as e:
        frappe.log_error(frappe.get_traceback(), "Stock Check Error")
        frappe.throw(_("Stock check failed: {0}").format(str(e)))
```
**Why:** Silent error swallowing makes production debugging impossible.

---

## 7. Not Checking Database Results

```python
# ❌ WRONG — Crashes on missing record
def validate(self):
    customer = frappe.get_doc("Customer", self.customer)  # DoesNotExistError!
    prices = frappe.db.sql("SELECT price FROM tabPrices WHERE item=%s", self.item)
    self.price = prices[0][0]  # IndexError if empty!

# ✅ CORRECT
def validate(self):
    if not frappe.db.exists("Customer", self.customer):
        frappe.throw(_("Customer not found"))
    customer = frappe.get_doc("Customer", self.customer)

    prices = frappe.db.sql("SELECT price FROM tabPrices WHERE item=%s", self.item)
    self.price = prices[0][0] if prices else 0
```
**Why:** ALWAYS verify data exists before accessing it.

---

## 8. Not Isolating Errors in on_update/on_cancel

```python
# ❌ WRONG — First failure stops all operations
def on_update(self):
    self.send_email()          # If this fails...
    self.sync_to_crm()        # ...this never runs
    self.update_dashboard()   # ...neither does this

# ✅ CORRECT — Each operation isolated
def on_update(self):
    operations = [
        (self.send_email, "Email"),
        (self.sync_to_crm, "CRM sync"),
        (self.update_dashboard, "Dashboard"),
    ]
    errors = []
    for op, label in operations:
        try:
            op()
        except Exception:
            errors.append(label)
            frappe.log_error(frappe.get_traceback(), f"{label} Error")

    if errors:
        frappe.msgprint(
            _("Saved. Some operations failed: {0}").format(", ".join(errors)),
            indicator="orange"
        )
```
**Why:** Independent post-save operations should not block each other.

---

## 9. Exposing Technical Errors to Users

```python
# ❌ WRONG
except Exception as e:
    frappe.throw(str(e))  # Stack trace to user!
    frappe.throw(frappe.get_traceback())  # Even worse!

# ✅ CORRECT
except requests.Timeout:
    frappe.throw(_("Service timed out. Please try again."))
except ConnectionError:
    frappe.throw(_("Could not connect to external service."))
except Exception as e:
    frappe.log_error(frappe.get_traceback(), "External Service Error")
    frappe.throw(_("Service error. Please contact support."))
```
**Why:** Technical details confuse users and may expose sensitive information.

---

## 10. Broad Exception Handling Without Specificity

```python
# ❌ WRONG — All errors get same vague message
try:
    self.check_customer()
    self.validate_items()
    self.calculate_totals()
except Exception:
    frappe.throw(_("Validation failed"))

# ✅ CORRECT — Specific handling per operation
try:
    self.check_customer()
except frappe.DoesNotExistError:
    frappe.throw(_("Customer not found"))

try:
    self.validate_items()
except frappe.ValidationError:
    raise  # Re-raise with original message

self.calculate_totals()  # Let errors propagate naturally
```
**Why:** Specific exception handling gives better error messages and debugging.

---

## 11. Heavy Operations in validate

```python
# ❌ WRONG — 30-second API call blocks save
def validate(self):
    self.sync_to_external_api()  # Slow!
    self.generate_pdf()          # Slow!

# ✅ CORRECT — Queue heavy work
def validate(self):
    self.validate_fields()  # Fast validation only

def on_update(self):
    frappe.enqueue("myapp.tasks.sync_and_generate",
        doctype=self.doctype, name=self.name, queue="long")
```
**Why:** Heavy operations in `validate` make the UI unresponsive and can timeout.

---

## 12. Missing Translation Wrapper

```python
# ❌ WRONG — Not translatable
frappe.throw("Customer is required")
frappe.msgprint("Order saved")

# ✅ CORRECT
frappe.throw(_("Customer is required"))
frappe.msgprint(_("Order saved"))
```
**Why:** Without `_()`, messages are English-only regardless of user's language.

---

## 13. Throwing in on_cancel Cleanup

```python
# ❌ WRONG — First throw stops remaining cleanup
def on_cancel(self):
    self.reverse_stock()   # If this throws...
    self.reverse_gl()      # ...this never runs!

# ✅ CORRECT — Collect errors, try all operations
def on_cancel(self):
    errors = []
    for op, label in [
        (self.reverse_stock, "Stock"),
        (self.reverse_gl, "GL entries"),
    ]:
        try:
            op()
        except Exception as e:
            errors.append(f"{label}: {str(e)}")
            frappe.log_error(frappe.get_traceback(), f"{label} Error")

    if errors:
        frappe.msgprint(
            _("Cancelled with issues: {0}").format("<br>".join(errors)),
            indicator="orange"
        )
```
**Why:** Cancel cleanup should attempt ALL operations, not stop at the first failure.

---

## 14. Not Using Flags for Recursion Guard

```python
# ❌ WRONG — Updating linked doc triggers back-update loop
def on_update(self):
    if self.quotation:
        q = frappe.get_doc("Quotation", self.quotation)
        q.db_set("status", "Ordered")  # May trigger Quotation.on_update → back to here

# ✅ CORRECT
def on_update(self):
    if self.flags.get("skip_linked_update"):
        return
    if self.quotation:
        q = frappe.get_doc("Quotation", self.quotation)
        q.flags.skip_linked_update = True
        q.db_set("status", "Ordered")
```
**Why:** Cross-document updates can create circular hook triggers without recursion guards.

---

## Pre-Deploy Checklist

- [ ] `super()` called in ALL overridden hooks
- [ ] No `self.save()` in any hook
- [ ] No `frappe.db.commit()` calls
- [ ] `on_update` changes use `db_set()`
- [ ] All validation in `before_submit`, not `on_submit`
- [ ] All exceptions logged with `frappe.log_error()`
- [ ] Error messages use `_()` wrapper
- [ ] None/empty values handled safely
- [ ] Post-save operations isolated in try/except
- [ ] Heavy operations enqueued, not inline
- [ ] Specific exceptions caught before generic `Exception`
- [ ] Recursion guards (`flags`) on cross-document updates
- [ ] `on_cancel` operations isolated — don't stop on first failure

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/errors/frappe-errors-controllers/references/anti-patterns.md*
