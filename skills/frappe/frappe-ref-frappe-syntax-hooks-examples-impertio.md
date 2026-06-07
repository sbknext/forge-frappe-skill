---
title: Frappe Ref — Frappe Syntax: Hooks: Examples (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'hooks', 'reference', 'examples']
source: Impertio
---

# Complete hooks.py Examples

Working examples for each hook category. For `doc_events` examples, see the
**frappe-syntax-hooks-events** skill.

---

## Minimal hooks.py

```python
app_name = "myapp"
app_title = "My App"
app_publisher = "My Company"
app_description = "My custom ERPNext app"
app_email = "info@mycompany.com"
app_license = "MIT"
```

---

## Standard Business App hooks.py

```python
app_name = "myapp"
app_title = "My App"
app_publisher = "My Company"
app_description = "Custom ERPNext extensions"
app_email = "info@mycompany.com"
app_license = "MIT"

required_apps = ["erpnext"]

# ============================================================
# Frontend Assets
# ============================================================
app_include_js = "/assets/myapp/js/myapp.min.js"
app_include_css = "/assets/myapp/css/myapp.min.css"

doctype_js = {
    "Sales Invoice": "public/js/sales_invoice.js",
    "Customer": "public/js/customer.js"
}

doctype_list_js = {
    "Sales Invoice": "public/js/sales_invoice_list.js"
}

# ============================================================
# Scheduled Tasks
# ============================================================
scheduler_events = {
    "daily": [
        "myapp.tasks.send_daily_digest",
        "myapp.tasks.cleanup_old_logs"
    ],
    "daily_long": [
        "myapp.tasks.sync_external_system"
    ],
    "cron": {
        "0 9 * * 1-5": ["myapp.tasks.weekday_morning_report"],
        "0 17 * * 5": ["myapp.tasks.weekly_summary"]
    }
}

# ============================================================
# Client-Side Data
# ============================================================
extend_bootinfo = "myapp.boot.extend_boot"
notification_config = "myapp.notifications.get_config"

# ============================================================
# Custom Permissions
# ============================================================
permission_query_conditions = {
    "Sales Invoice": "myapp.permissions.si_query_conditions"
}
has_permission = {
    "Sales Invoice": "myapp.permissions.si_has_permission"
}

# ============================================================
# Fixtures
# ============================================================
fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My App"]]},
    {"dt": "Property Setter", "filters": [["module", "=", "My App"]]},
    {"dt": "Client Script", "filters": [["module", "=", "My App"]]},
    {"dt": "Role", "filters": [["name", "like", "MyApp%"]]}
]

# ============================================================
# Install / Migrate
# ============================================================
after_install = "myapp.setup.after_install"
after_migrate = "myapp.setup.after_migrate"

# ============================================================
# Jinja Extensions
# ============================================================
jinja = {
    "methods": ["myapp.jinja_utils.get_customer_balance"],
    "filters": ["myapp.jinja_utils.format_iban"]
}
```

---

## Setup Module (myapp/setup.py)

```python
import frappe

def after_install():
    """Post-installation setup. No arguments."""
    create_default_roles()
    create_default_settings()

def create_default_roles():
    roles = ["MyApp User", "MyApp Manager"]
    for role in roles:
        if not frappe.db.exists("Role", role):
            frappe.get_doc({
                "doctype": "Role",
                "role_name": role
            }).insert()

def create_default_settings():
    if not frappe.db.exists("My App Settings"):
        frappe.get_doc({
            "doctype": "My App Settings",
            "enable_feature_x": 1
        }).insert()

def after_migrate():
    """Runs after every bench migrate. No arguments."""
    frappe.cache().delete_key("myapp_config")
```

---

## Boot Module (myapp/boot.py)

```python
import frappe

def extend_boot(bootinfo):
    """Inject app-specific data into frappe.boot."""
    bootinfo.myapp_version = frappe.get_module("myapp").__version__

    if frappe.session.user != "Guest":
        bootinfo.myapp_settings = get_user_settings()
        bootinfo.feature_flags = get_feature_flags()

    default_company = frappe.defaults.get_user_default("Company")
    if default_company:
        bootinfo.company_config = frappe.db.get_value(
            "Company",
            default_company,
            ["default_currency", "country"],
            as_dict=True
        )

def get_user_settings():
    user = frappe.session.user
    return {
        "dashboard_layout": frappe.db.get_value(
            "User", user, "dashboard_layout"
        ) or "default"
    }

def get_feature_flags():
    settings = frappe.get_single("My App Settings")
    return {
        "new_dashboard": settings.enable_new_dashboard,
        "beta_features": settings.enable_beta
    }
```

---

## Tasks Module (myapp/tasks.py)

```python
import frappe
from frappe.utils import today, add_days

def send_daily_digest():
    """Daily digest for sales team. No arguments."""
    users = frappe.get_all(
        "User",
        filters={"enabled": 1, "user_type": "System User"},
        fields=["name", "email"]
    )
    for user in users:
        if "Sales User" in frappe.get_roles(user.name):
            digest = compile_digest(user.name)
            if digest:
                frappe.sendmail(
                    recipients=[user.email],
                    subject=f"Daily Sales Digest - {today()}",
                    message=digest
                )

def cleanup_old_logs():
    """Remove logs older than 30 days. No arguments."""
    cutoff = add_days(today(), -30)
    frappe.db.delete("Activity Log", {"creation": ["<", cutoff]})
    frappe.db.commit()

def sync_external_system():
    """Long running sync. Use daily_long. No arguments."""
    records = frappe.get_all(
        "Sync Queue",
        filters={"status": "Pending"},
        limit=1000
    )
    for record in records:
        try:
            process_sync(record.name)
            frappe.db.set_value("Sync Queue", record.name, "status", "Completed")
        except Exception:
            frappe.db.set_value("Sync Queue", record.name, "status", "Failed")
            frappe.log_error(
                title=f"Sync Failed: {record.name}",
                message=frappe.get_traceback()
            )
        frappe.db.commit()  # Commit per record

def weekday_morning_report():
    """Cron: 0 9 * * 1-5. No arguments."""
    yesterday = add_days(today(), -1)
    report = generate_daily_report(yesterday)
    frappe.sendmail(
        recipients=["management@mycompany.com"],
        subject=f"Daily Business Report - {yesterday}",
        message=report
    )
```

---

## Permissions Module (myapp/permissions.py)

```python
import frappe

def si_query_conditions(user):
    """Filter Sales Invoices in list view."""
    if not user:
        user = frappe.session.user

    if user == "Administrator":
        return ""

    roles = frappe.get_roles(user)

    if "Accounts Manager" in roles:
        return ""

    if "Accounts User" in roles:
        company = frappe.defaults.get_user_default("Company")
        if company:
            return f"`tabSales Invoice`.company = {frappe.db.escape(company)}"

    if "Sales User" in roles:
        return f"`tabSales Invoice`.owner = {frappe.db.escape(user)}"

    return "1=0"

def si_has_permission(doc, user=None, permission_type=None):
    """Document-level permission for Sales Invoice."""
    if not user:
        user = frappe.session.user

    if permission_type == "write" and doc.status == "Closed":
        return False

    return None
```

---

## Jinja Utilities (myapp/jinja_utils.py)

```python
import frappe

def get_customer_balance(customer):
    """Usage in template: {{ get_customer_balance(doc.customer) }}"""
    return frappe.db.get_value(
        "Customer", customer, "outstanding_amount"
    ) or 0

def format_iban(value):
    """Usage in template: {{ bank_account|format_iban }}"""
    if not value:
        return ""
    return " ".join([value[i:i+4] for i in range(0, len(value), 4)])
```

---

## Website Hooks Example

```python
# hooks.py additions for website/portal
website_route_rules = [
    {"from_route": "/my-orders/<name>", "to_route": "My Order"}
]

portal_menu_items = [
    {"title": "My Orders", "route": "/my-orders", "role": "Customer"}
]

role_home_page = {
    "Customer": "my-orders",
    "Supplier": "my-rfqs"
}

update_website_context = "myapp.context.update_context"
```

```python
# myapp/context.py
def update_context(context):
    """Add dynamic values to website context."""
    context.app_version = "1.0.0"
    context.support_email = "support@mycompany.com"
```

---

## DocType Override Example (v14/v15)

```python
# hooks.py
override_doctype_class = {
    "Sales Invoice": "myapp.overrides.CustomSalesInvoice"
}
```

```python
# myapp/overrides.py
from erpnext.accounts.doctype.sales_invoice.sales_invoice import SalesInvoice

class CustomSalesInvoice(SalesInvoice):
    def validate(self):
        super().validate()  # ALWAYS call super() first
        self.validate_customer_status()

    def validate_customer_status(self):
        status = frappe.db.get_value("Customer", self.customer, "status")
        if status == "Blocked":
            frappe.throw(f"Cannot create invoice for blocked customer {self.customer}")
```

---

## DocType Extension Example (v16+)

```python
# hooks.py
extend_doctype_class = {
    "Address": ["myapp.extensions.AddressMixin"]
}
```

```python
# myapp/extensions.py
import re
import frappe
from frappe.model.document import Document

class AddressMixin(Document):
    def validate(self):
        super().validate()
        if self.country == "Netherlands" and self.pincode:
            if not re.match(r'^\d{4}\s?[A-Z]{2}$', self.pincode):
                frappe.throw("Invalid Dutch postal code")
```

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-hooks/references/examples.md*
