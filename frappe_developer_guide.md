# Comprehensive Frappe Developer Guide: Syntax and Core Concepts

> **Download Instructions**: Right-click on this artifact and select "Save as" or copy the content to a `.md` file on your local machine for offline reference.

## Table of Contents
1. [Core Architecture Overview](#core-architecture-overview)
2. [Essential Objects and Variables](#essential-objects-and-variables)
3. [Document Lifecycle](#document-lifecycle)
4. [Client-Side vs Server-Side](#client-side-vs-server-side)
5. [Hooks and Events](#hooks-and-events)
6. [Database Operations](#database-operations)
7. [Common Patterns and Syntax](#common-patterns-and-syntax)
8. [Comparison with Laravel/Next.js](#comparison-with-laravelnextjs)
9. [Best Practices](#best-practices)

---

## Core Architecture Overview

### Framework Structure
Frappe follows a **Model-View-Controller (MVC)** pattern but with its own terminology:

```
Laravel Term     →  Frappe Term
Model           →  DocType
Controller      →  Controller/Server Script
View            →  Form/List/Report
Migration       →  Patch
Middleware      →  Hooks
```

### Key Components
- **DocType**: Like Laravel Models - defines document structure and behavior
- **Document**: Instance of a DocType (like a Model instance)
- **Form**: UI for editing documents (like Laravel views/forms)
- **Server Script**: Python code that runs on the server
- **Client Script**: JavaScript code that runs in the browser

---

## Essential Objects and Variables

### 1. `cur_frm` - Current Form Object
**What it is**: The active form instance in the browser
**Laravel equivalent**: Like `$this` in a Laravel Form Request or View

```javascript
// Access current form
cur_frm.doc           // Current document data
cur_frm.fields_dict   // All form fields
cur_frm.is_new()      // Check if document is new
cur_frm.save()        // Save the document
cur_frm.refresh()     // Refresh the form

// Example: Hide field based on condition
if (cur_frm.doc.status === 'Cancelled') {
    cur_frm.set_df_property('amount', 'hidden', 1);
}
```

### 2. `doc` - Current Document
**What it is**: The document data object
**Laravel equivalent**: Like `$model` or `$user` in Laravel

```javascript
// In Client Script
frappe.ui.form.on('Sales Order', {
    refresh: function(frm) {
        console.log(frm.doc.name);        // Document ID
        console.log(frm.doc.customer);    // Field value
        console.log(frm.doc.total);       // Another field
    }
});
```

```python
# In Server Script/Python
def validate(doc, method):
    if doc.status == "Draft":
        doc.total_amount = sum(item.amount for item in doc.items)
```

### 3. `frm` - Form Parameter
**What it is**: The form object passed to event handlers
**Usage**: Same as `cur_frm` but passed as parameter

```javascript
frappe.ui.form.on('Sales Order', {
    onload: function(frm) {
        // frm is the same as cur_frm in this context
        frm.set_value('date', frappe.datetime.get_today());
    }
});
```

### 4. `frappe` - Global Object
**What it is**: Main framework object (like Laravel's `App` facade)
**Laravel equivalent**: Like Laravel's global helpers or facades

```javascript
// Common frappe methods
frappe.msgprint("Hello World");                    // Show message
frappe.throw("Error occurred");                    // Throw error
frappe.db.get_value('User', 'Administrator', 'email'); // Database query
frappe.call({                                      // API call
    method: 'my_app.api.get_data',
    callback: function(r) {
        console.log(r.message);
    }
});
```

### 5. `locals` - Document Cache
**What it is**: Client-side cache of documents
**Usage**: Access any loaded document

```javascript
// Access any loaded document
locals['Sales Order']['SO-2024-001']  // Specific document
locals['Sales Order'][cur_frm.doc.name] // Current document
```

---

## Document Lifecycle

### States and Methods
```javascript
// Document states
doc.docstatus === 0  // Draft
doc.docstatus === 1  // Submitted
doc.docstatus === 2  // Cancelled

// Common methods
cur_frm.save()       // Save document
cur_frm.submit()     // Submit document
cur_frm.cancel()     // Cancel document
cur_frm.amend()      // Amend cancelled document
```

### Lifecycle Events (Client-Side)
```javascript
frappe.ui.form.on('DocType Name', {
    // When form loads
    onload: function(frm) {
        // Initialize form
    },
    
    // When form refreshes
    refresh: function(frm) {
        // Update UI, set permissions
    },
    
    // Before save
    before_save: function(frm) {
        // Validation before save
    },
    
    // After save
    after_save: function(frm) {
        // Actions after successful save
    },
    
    // When field changes
    field_name: function(frm) {
        // Handle field change
    }
});
```

### Lifecycle Events (Server-Side)
```python
# In DocType Python file or Server Script
def validate(self):
    # Before save validation
    pass

def before_save(self):
    # Just before saving
    pass

def after_insert(self):
    # After document creation
    pass

def on_submit(self):
    # When document is submitted
    pass

def on_cancel(self):
    # When document is cancelled
    pass
```

---

## Client-Side vs Server-Side

### Client-Side (JavaScript)
- Runs in browser
- UI interactions, field changes
- Form validation, custom buttons
- Real-time updates

```javascript
// Client Script example
frappe.ui.form.on('Sales Order', {
    customer: function(frm) {
        // Runs when customer field changes
        if (frm.doc.customer) {
            frappe.call({
                method: 'erpnext.selling.doctype.sales_order.sales_order.get_customer_details',
                args: {
                    customer: frm.doc.customer
                },
                callback: function(r) {
                    if (r.message) {
                        frm.set_value('customer_address', r.message.address);
                    }
                }
            });
        }
    }
});
```

### Server-Side (Python)
- Runs on server
- Database operations, business logic
- Email sending, file operations
- Background jobs

```python
# Server Script example
import frappe

@frappe.whitelist()
def get_customer_details(customer):
    """Get customer details"""
    customer_doc = frappe.get_doc('Customer', customer)
    return {
        'address': customer_doc.customer_primary_address,
        'contact': customer_doc.customer_primary_contact
    }

def validate(doc, method):
    """Validate document before save"""
    if doc.total < 0:
        frappe.throw("Total cannot be negative")
```

---

## Hooks and Events

### Client Script Events
```javascript
// Form events
frappe.ui.form.on('DocType', {
    refresh: function(frm) {},
    setup: function(frm) {},
    onload: function(frm) {},
    before_save: function(frm) {},
    after_save: function(frm) {},
    before_submit: function(frm) {},
    on_submit: function(frm) {},
    before_cancel: function(frm) {},
    after_cancel: function(frm) {}
});

// Field events
frappe.ui.form.on('DocType', {
    field_name: function(frm) {
        // When field changes
    }
});

// Child table events
frappe.ui.form.on('Child DocType', {
    items_add: function(frm, cdt, cdn) {
        // When row added to child table
    },
    items_remove: function(frm, cdt, cdn) {
        // When row removed from child table
    }
});
```

### Server Script Events
```python
# Document events
def validate(doc, method):
    pass

def before_save(doc, method):
    pass

def after_insert(doc, method):
    pass

def on_submit(doc, method):
    pass

def on_cancel(doc, method):
    pass

def before_delete(doc, method):
    pass
```

---

## Database Operations

### Client-Side Database Calls
```javascript
// Get single value
frappe.db.get_value('Customer', 'CUST-001', 'customer_name')
    .then(r => {
        console.log(r.message.customer_name);
    });

// Get multiple values
frappe.db.get_value('Customer', 'CUST-001', ['customer_name', 'territory'])
    .then(r => {
        console.log(r.message);
    });

// Get list of documents
frappe.db.get_list('Customer', {
    filters: {
        'territory': 'India'
    },
    fields: ['name', 'customer_name'],
    limit: 10
}).then(r => {
    console.log(r);
});

// Set value
frappe.db.set_value('Customer', 'CUST-001', 'customer_name', 'New Name');
```

### Server-Side Database Operations
```python
# Get document
doc = frappe.get_doc('Customer', 'CUST-001')

# Get single value
customer_name = frappe.db.get_value('Customer', 'CUST-001', 'customer_name')

# Get multiple values
values = frappe.db.get_value('Customer', 'CUST-001', ['customer_name', 'territory'])

# Get list
customers = frappe.db.get_list('Customer', 
    filters={'territory': 'India'},
    fields=['name', 'customer_name']
)

# Raw SQL
data = frappe.db.sql("""
    SELECT name, customer_name 
    FROM `tabCustomer` 
    WHERE territory = %s
""", ('India',), as_dict=True)

# Set value
frappe.db.set_value('Customer', 'CUST-001', 'customer_name', 'New Name')

# Insert document
doc = frappe.get_doc({
    'doctype': 'Customer',
    'customer_name': 'Test Customer',
    'territory': 'India'
})
doc.insert()
```

---

## Common Patterns and Syntax

### 1. Setting Field Values
```javascript
// Client-side
frm.set_value('field_name', 'value');
frm.set_value('date', frappe.datetime.get_today());

// Multiple values
frm.set_value({
    'field1': 'value1',
    'field2': 'value2'
});
```

### 2. Field Properties
```javascript
// Hide/show fields
frm.set_df_property('field_name', 'hidden', 1);  // Hide
frm.set_df_property('field_name', 'hidden', 0);  // Show

// Read-only
frm.set_df_property('field_name', 'read_only', 1);

// Required
frm.set_df_property('field_name', 'reqd', 1);

// Options for select field
frm.set_df_property('field_name', 'options', 'Option1\nOption2\nOption3');
```

### 3. Custom Buttons
```javascript
// Add custom button
frm.add_custom_button('Button Name', function() {
    // Button action
    frappe.msgprint('Button clicked!');
});

// Add button to specific group
frm.add_custom_button('Action', function() {
    // Action
}, 'Group Name');

// Remove button
frm.remove_custom_button('Button Name');
```

### 4. Filters and Queries
```javascript
// Set filter on field
frm.set_query('customer', function() {
    return {
        filters: {
            'territory': 'India'
        }
    };
});

// Dynamic filter
frm.set_query('item_code', 'items', function(doc, cdt, cdn) {
    return {
        filters: {
            'item_group': 'Products'
        }
    };
});
```

### 5. Child Table Operations
```javascript
// Add row to child table
let row = frm.add_child('items');
row.item_code = 'ITEM-001';
row.qty = 1;
frm.refresh_field('items');

// Get child table data
frm.doc.items.forEach(function(item) {
    console.log(item.item_code, item.qty);
});

// Clear child table
frm.clear_table('items');
frm.refresh_field('items');
```

---

## Comparison with Laravel/Next.js

### Laravel vs Frappe Comparison

| Laravel | Frappe | Purpose |
|---------|--------|---------|
| `Model::create()` | `frappe.get_doc().insert()` | Create record |
| `Model::find()` | `frappe.get_doc()` | Get single record |
| `Model::where()` | `frappe.get_list()` | Query records |
| `$request->validate()` | `validate()` method | Validation |
| `Route::get()` | `@frappe.whitelist()` | API endpoints |
| `Blade templates` | Jinja2 templates | Templating |
| `Eloquent relationships` | Link fields | Relationships |
| `Middleware` | Hooks | Middleware |
| `Artisan commands` | Bench commands | CLI |

### Next.js vs Frappe Frontend

| Next.js | Frappe | Purpose |
|---------|--------|---------|
| `useState()` | `frm.set_value()` | State management |
| `useEffect()` | Form events | Side effects |
| `fetch()` | `frappe.call()` | API calls |
| `router.push()` | `frappe.set_route()` | Navigation |
| `props` | `frm.doc` | Data passing |
| `components` | Custom forms | UI components |

---

## Best Practices

### 1. Client Script Best Practices
```javascript
// Use specific event handlers
frappe.ui.form.on('DocType', {
    // Good: Specific field handler
    customer: function(frm) {
        // Handle customer change
    },
    
    // Avoid: Generic refresh for field changes
    refresh: function(frm) {
        // Don't put field-specific logic here
    }
});

// Cache expensive operations
frappe.ui.form.on('DocType', {
    setup: function(frm) {
        // Cache data that doesn't change
        frm.custom_data = {};
    }
});
```

### 2. Server Script Best Practices
```python
# Use proper error handling
def validate(doc, method):
    try:
        # Validation logic
        if not doc.customer:
            frappe.throw("Customer is required")
    except Exception as e:
        frappe.log_error(f"Validation error: {str(e)}")
        frappe.throw("An error occurred during validation")

# Use database transactions
def create_related_documents(doc, method):
    try:
        # Create related documents
        related_doc = frappe.get_doc({
            'doctype': 'Related DocType',
            'reference': doc.name
        })
        related_doc.insert()
        frappe.db.commit()
    except Exception:
        frappe.db.rollback()
        raise
```

### 3. Performance Tips
```javascript
// Batch database operations
frappe.call({
    method: 'my_app.api.batch_update',
    args: {
        'documents': frm.doc.items
    },
    callback: function(r) {
        // Handle response
    }
});

// Use debouncing for frequent operations
let timeout;
frappe.ui.form.on('DocType', {
    field_name: function(frm) {
        clearTimeout(timeout);
        timeout = setTimeout(() => {
            // Actual operation
        }, 300);
    }
});
```

### 4. Code Organization
```javascript
// Organize code in modules
frappe.ui.form.on('Sales Order', {
    onload: function(frm) {
        sales_order_controller.setup_form(frm);
    }
});

// Create controller object
let sales_order_controller = {
    setup_form: function(frm) {
        this.setup_filters(frm);
        this.setup_buttons(frm);
    },
    
    setup_filters: function(frm) {
        // Filter setup
    },
    
    setup_buttons: function(frm) {
        // Button setup
    }
};
```

---

## Common Gotchas and Solutions

### 1. Timing Issues
```javascript
// Problem: Field not available immediately
frappe.ui.form.on('DocType', {
    onload: function(frm) {
        // This might not work
        frm.set_value('field', 'value');
    }
});

// Solution: Use setup or refresh
frappe.ui.form.on('DocType', {
    setup: function(frm) {
        // Form is fully loaded
        frm.set_value('field', 'value');
    }
});
```

### 2. Child Table Issues
```javascript
// Problem: Child table not refreshing
let row = frm.add_child('items');
row.item_code = 'ITEM-001';
// Table doesn't update visually

// Solution: Always refresh field
frm.refresh_field('items');
```

### 3. Permission Issues
```javascript
// Check permissions before operations
if (frm.perm[0].write) {
    frm.set_value('field', 'value');
}
```

This comprehensive guide should help you understand Frappe's syntax and patterns. The key is to think of Frappe as having a client-server architecture where the client (browser) and server (Python) communicate through a well-defined API, similar to how Next.js communicates with a backend API.