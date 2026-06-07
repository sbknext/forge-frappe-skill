---
title: Frappe Ref — Frappe Syntax: Controllers: Syntax (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'controllers', 'reference', 'syntax']
source: Impertio
---

# Controller Class Syntax Reference

Exact syntax for Document Controller classes in Frappe v14-v16.

---

## Minimal Controller

```python
# {app}/{module}/doctype/{doctype_snake}/{doctype_snake}.py
import frappe
from frappe.model.document import Document

class MyDocType(Document):
    pass
```

EVERY DocType MUST have a controller file, even if empty.

---

## Standard Controller Template

```python
import frappe
from frappe import _
from frappe.model.document import Document

class SalesOrder(Document):
    # ---- NAMING ----
    def autoname(self):
        self.name = f"SO-{self.company_abbr}-{frappe.utils.now_datetime().year}"

    # ---- VALIDATION ----
    def before_validate(self):
        self.set_defaults()

    def validate(self):
        self.validate_items()
        self.calculate_totals()

    def before_save(self):
        self.set_status()

    # ---- INSERT ONLY ----
    def before_insert(self):
        self.created_by = frappe.session.user

    def after_insert(self):
        self.add_comment("Created", f"Created by {frappe.session.user}")

    # ---- POST-SAVE ----
    def on_update(self):
        self.notify_relevant_users()

    def on_change(self):
        pass  # MUST be idempotent

    # ---- SUBMIT/CANCEL ----
    def before_submit(self):
        self.check_approval()

    def on_submit(self):
        self.create_ledger_entries()

    def before_cancel(self):
        self.check_linked_docs()

    def on_cancel(self):
        self.reverse_ledger_entries()

    # ---- UPDATE AFTER SUBMIT ----
    def before_update_after_submit(self):
        self.validate_allowed_changes()

    def on_update_after_submit(self):
        self.log_amendment()

    # ---- DELETE ----
    def on_trash(self):
        self.cleanup_related_data()

    def after_delete(self):
        pass

    # ---- RENAME ----
    def before_rename(self, old_name, new_name, merge=False):
        pass

    def after_rename(self, old_name, new_name, merge=False):
        pass

    # ---- PRINT ----
    def before_print(self, print_settings=None):
        self.print_date = frappe.utils.today()

    # ---- WHITELISTED (callable from JS) ----
    @frappe.whitelist()
    def recalculate(self):
        self.calculate_totals()
        return {"total": self.total}

    # ---- PRIVATE METHODS ----
    def validate_items(self):
        if not self.items:
            frappe.throw(_("Items are required"))

    def calculate_totals(self):
        self.total = sum(item.amount for item in self.items)
```

---

## Type Annotations [v15+]

```python
import frappe
from frappe.model.document import Document

class Person(Document):
    # begin: auto-generated types
    # This block is auto-generated. Do not modify.
    from typing import TYPE_CHECKING
    if TYPE_CHECKING:
        from frappe.types import DF

        first_name: DF.Data
        last_name: DF.Data
        email: DF.Data
        birth_date: DF.Date
        company: DF.Link
        is_active: DF.Check
        notes: DF.TextEditor
        items: DF.Table["PersonItem"]
    # end: auto-generated types

    def validate(self):
        if not self.first_name:
            frappe.throw(_("First name is required"))
```

Enable in hooks.py: `export_python_type_annotations = True`

---

## Tree DocType Controller

```python
import frappe
from frappe import _
from frappe.utils.nestedset import NestedSet

class Department(NestedSet):
    nsm_parent_field = "parent_department"

    def validate(self):
        if self.parent_department == self.name:
            frappe.throw(_("Cannot be own parent"))

    def on_update(self):
        super().on_update()  # ALWAYS call super for NestedSet
```

---

## Virtual DocType Controller

```python
import frappe
from frappe.model.document import Document

class ExternalData(Document):
    """DocType with Is Virtual = 1. No database table."""

    def load_from_db(self):
        """Called by frappe.get_doc(). Load from external source."""
        data = external_api.get(self.name)
        super(Document, self).__init__(data)

    def db_insert(self, *args, **kwargs):
        """Called by doc.insert()."""
        external_api.create(self.as_dict())

    def db_update(self, *args, **kwargs):
        """Called by doc.save()."""
        external_api.update(self.name, self.as_dict())

    @staticmethod
    def get_list(args):
        """Return list for List View. MUST be @staticmethod."""
        return external_api.list(args.get("filters"), args.get("page_length", 20))

    @staticmethod
    def get_count(args):
        """Return count for pagination. MUST be @staticmethod."""
        return external_api.count(args.get("filters"))

    @staticmethod
    def get_stats(args):
        """Return stats for sidebar. MUST be @staticmethod."""
        return {}
```

---

## Controller Override Class

```python
# myapp/overrides/custom_sales_order.py
from erpnext.selling.doctype.sales_order.sales_order import SalesOrder

class CustomSalesOrder(SalesOrder):
    def validate(self):
        super().validate()  # ALWAYS call super first
        self.custom_validation()

    def on_submit(self):
        super().on_submit()  # ALWAYS call super
        self.custom_post_submit()
```

Register in hooks.py:
```python
override_doctype_class = {
    "Sales Order": "myapp.overrides.custom_sales_order.CustomSalesOrder"
}
```

---

## Extension Mixin [v16+]

```python
# myapp/extensions/audit_mixin.py
from frappe.model.document import Document

class AuditMixin(Document):
    def validate(self):
        super().validate()  # ALWAYS call super
        self.set_audit_fields()

    def set_audit_fields(self):
        if self.is_new():
            self.custom_created_by = frappe.session.user
        self.custom_last_modified_by = frappe.session.user
```

Register in hooks.py:
```python
extend_doctype_class = {
    "Sales Order": ["myapp.extensions.audit_mixin.AuditMixin"],
    "Purchase Order": ["myapp.extensions.audit_mixin.AuditMixin"]
}
```

---

## doc_events Handler Function

```python
# myapp/events/sales_order.py
import frappe
from frappe import _

def validate(doc, method=None):
    """EXACT signature: (doc, method=None). NEVER change this."""
    if doc.total > 100000:
        doc.requires_approval = 1

def on_submit(doc, method=None):
    frappe.enqueue(
        "myapp.tasks.sync_order",
        queue="short",
        doc_name=doc.name,
        enqueue_after_commit=True
    )
```

Register in hooks.py:
```python
doc_events = {
    "Sales Order": {
        "validate": "myapp.events.sales_order.validate",
        "on_submit": "myapp.events.sales_order.on_submit"
    }
}
```

---

## Whitelisted Method Syntax

```python
class SalesOrder(Document):
    @frappe.whitelist()
    def method_name(self, param1, param2="default"):
        """Callable from JS: frm.call('method_name', {param1: 'value'})"""
        # self is the document instance
        # Return value is available as r.message in JS
        return {"result": "value"}
```

Client-side call:
```javascript
frm.call('method_name', { param1: 'value' }).then(r => {
    console.log(r.message.result);  // "value"
});
```

---

## Import Conventions

```python
# ALWAYS import
import frappe
from frappe import _

# Controller base class
from frappe.model.document import Document

# Tree DocType base
from frappe.utils.nestedset import NestedSet

# Naming utilities
from frappe.model.naming import getseries, make_autoname

# Common utilities
from frappe.utils import (
    cint, cstr, flt,
    now, now_datetime, today,
    getdate, get_datetime,
    format_date, format_datetime
)
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-controllers/references/syntax.md*
