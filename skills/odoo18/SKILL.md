# Odoo 18 Development Skills

This document provides conventions and patterns for writing code indistinguishable from the Odoo 18 codebase, based on analysis of the `addons/project` module and core framework.

## Project Structure

```
addon_name/
├── __init__.py           # Module imports
├── __manifest__.py       # Module metadata
├── controllers/          # HTTP controllers
│   ├── __init__.py
│   └── portal.py
├── models/               # Business logic
│   ├── __init__.py
│   └── model_name.py
├── views/                # XML views
│   └── model_views.xml
├── security/             # Access rights
│   ├── ir.model.access.csv
│   └── model_security.xml
├── tests/                # Unit tests
│   ├── __init__.py
│   └── test_feature.py
├── wizard/               # Transient models
├── report/               # Report definitions
├── data/                 # Demo/default data
├── static/               # Frontend assets
│   └── src/
│       ├── components/
│       ├── views/
│       └── js/
└── i18n/                 # Translations
```

## Python Code Style

### File Header

Every Python file must start with the copyright header (NO `coding: utf-8` in Odoo 18):

```python
# (c) Sozialinfo.ch See LICENSE file for full copyright and licensing details.
```

### Import Ordering

Imports are grouped in this order, separated by blank lines:

1. Standard library imports (alphabetically)
2. Third-party imports (alphabetically)
3. Odoo imports (`from odoo import ...`)
4. Odoo addon imports (`from odoo.addons...`)
5. Local imports (`from . import ...`)

```python
import ast
import json
from collections import defaultdict
from datetime import timedelta

from odoo import api, Command, fields, models, _
from odoo.addons.mail.tools.discuss import Store
from odoo.exceptions import UserError, ValidationError
from odoo.osv.expression import AND
from odoo.tools import get_lang, SQL

from .project_task import CLOSED_STATES
```

### Model Class Structure

```python
class Project(models.Model):
    _name = "project.project"
    _description = "Project"
    _inherit = [
        'portal.mixin',
        'mail.alias.mixin',
        'mail.thread',
    ]
    _order = "sequence, name, id"

    # Class attributes (no blank lines between these)
    _rating_satisfaction_days = 30
    _systray_view = 'activity'

    # Blank line, then helper methods used by field definitions
    def _default_stage_id(self):
        return self.env['project.project.stage'].search([], limit=1)

    # Blank line, then field definitions
    name = fields.Char("Name", required=True, tracking=True, translate=True)
    active = fields.Boolean(default=True, copy=False)

    # Blank line, then SQL constraints
    _sql_constraints = [
        ('project_date_greater', 'check(date >= date_start)',
         "The project's start date must be before its end date.")
    ]

    # Blank line, then compute/inverse/search methods
    @api.depends('milestone_ids', 'milestone_ids.is_reached')
    def _compute_milestone_count(self):
        ...

    # Blank line, then constraint methods
    @api.constrains('company_id', 'partner_id')
    def _ensure_company_consistency(self):
        ...

    # Blank line, then CRUD overrides
    def create(self, vals):
        ...

    # Blank line, then business methods
    def action_view_tasks(self):
        ...
```

### Field Definitions

- String parameter first, then other parameters
- Use `string=` explicitly only when different from field name
- Group related parameters, keep line length reasonable

```python
# Simple field
name = fields.Char("Name", required=True, tracking=True, translate=True)

# Field with lambda default
label_tasks = fields.Char(
    string='Use Tasks as',
    default=lambda s: s.env._('Tasks'),
    translate=True,
    help="Name used to refer to the tasks"
)

# Selection field
privacy_visibility = fields.Selection([
    ('followers', 'Invited internal users (private)'),
    ('employees', 'All internal users'),
    ('portal', 'Invited portal users and all internal users (public)'),
], string='Visibility', required=True, default='portal', tracking=True)

# Many2one with domain
partner_id = fields.Many2one(
    'res.partner', string='Customer',
    domain="['|', ('company_id', '=?', company_id), ('company_id', '=', False)]"
)

# Computed field with search
is_favorite = fields.Boolean(
    compute='_compute_is_favorite',
    readonly=False,
    search='_search_is_favorite',
    compute_sudo=True,
    string='Show Project on Dashboard',
    export_string_translation=False,
)

# Computed field with inverse (read/write computed field)
is_pinned = fields.Boolean(
    "Pinned",
    compute='_compute_is_pinned',
    inverse='_inverse_is_pinned',
    readonly=False,
)
```

### SQL Constraints

SQL constraint messages are **plain strings** without translation:

```python
_sql_constraints = [
    ('name_project_unique', 'unique(name, project_id)',
     'A note with this title already exists in this project.'),
    ('recurring_task_has_no_parent',
     'CHECK (NOT (recurring_task IS TRUE AND parent_id IS NOT NULL))',
     "A subtask cannot be recurrent."),
]
```

### Method Decorators

```python
# Compute method
@api.depends('milestone_ids', 'milestone_ids.is_reached')
def _compute_milestone_count(self):
    ...

# Compute with context
@api.depends_context('company')
@api.depends('company_id')
def _compute_currency_id(self):
    ...

# Model method (no self recordset)
@api.model
def _search_is_favorite(self, operator, value):
    ...

# Onchange
@api.onchange('company_id')
def _onchange_company_id(self):
    ...

# Create multi
@api.model_create_multi
def create(self, vals_list):
    ...

# Ondelete constraint
@api.ondelete(at_uninstall=False)
def _unlink_if_no_tasks(self):
    ...
```

### Error Handling

```python
from odoo.exceptions import UserError, ValidationError

# User-facing errors with _()
raise UserError(_('The project and the associated partner must be linked to the same company.'))

# ValidationError for constraint violations
raise ValidationError(_('This task has sub-tasks, so it cannot be private.'))

# NotImplementedError for unsupported operations
raise NotImplementedError(_('Operation not supported'))

# Formatted error messages
raise UserError(_(
    "This project is associated with %(project_company)s, whereas the selected stage belongs to %(stage_company)s.",
    project_company=project.company_id.name,
    stage_company=project.stage_id.company_id.name
))
```

### CRUD Patterns

```python
@api.model_create_multi
def create(self, vals_list):
    # Modify context for create
    self = self.with_context(mail_create_nosubscribe=True)
    # Pre-process vals_list
    for vals in vals_list:
        if vals.pop('is_favorite', False):
            vals['favorite_user_ids'] = [self.env.uid]
    return super().create(vals_list)

def write(self, vals):
    # Handle special fields before super
    if 'is_favorite' in vals:
        self._set_favorite_user_ids(vals.pop('is_favorite'))

    res = super().write(vals)

    # Post-processing after super
    if 'active' in vals:
        self.mapped('tasks').write({'active': vals['active']})
    return res

def unlink(self):
    # Cleanup before unlink
    self.tasks.unlink()
    return super().unlink()

def copy(self, default=None):
    default = dict(default or {})
    default['milestone_ids'] = False
    new_records = super().copy(default)
    return new_records

def copy_data(self, default=None):
    vals_list = super().copy_data(default=default)
    return [dict(vals, name=self.env._("%s (copy)", record.name))
            for record, vals in zip(self, vals_list)]
```

### Command Usage for Relational Fields

```python
from odoo import Command

# Link (add to many2many)
favorite_user_ids = [Command.link(self.env.uid)]

# Unlink (remove from many2many)
favorite_user_ids = [Command.unlink(self.env.uid)]

# Clear (remove all)
tag_ids = [Command.clear()]

# Set (replace all)
tag_ids = [Command.set(existing_tags.ids)]

# Create (create and link)
child_ids = [Command.create({'name': 'Subtask'})]

# Update
collaborator_ids = [Command.update(record.id, {'limited_access': True})]

# Delete
collaborator_ids = [Command.delete(record_id)]
```

### Read Group Pattern

```python
def _compute_task_count(self):
    tasks_count_by_project = dict(
        self.env['project.task'].with_context(
            active_test=any(project.active for project in self)
        )._read_group(
            [('project_id', 'in', self.ids), ('display_in_project', '=', True)],
            ['project_id'],
            ['__count']
        )
    )
    for project in self:
        project.task_count = tasks_count_by_project.get(project, 0)
```

## XML Views

### View Record Structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_model_form" model="ir.ui.view">
        <field name="name">model.form</field>
        <field name="model">model.name</field>
        <field name="arch" type="xml">
            <form string="Model">
                <header>
                    <button name="action_method" string="Action" type="object" class="oe_highlight"/>
                    <field name="state" widget="statusbar"/>
                </header>
                <sheet>
                    <div class="oe_button_box" name="button_box">
                        <button class="oe_stat_button" type="object" name="action_view_items" icon="fa-check">
                            <field name="item_count" widget="statinfo"/>
                        </button>
                    </div>
                    <widget name="web_ribbon" title="Archived" bg_color="text-bg-danger" invisible="active"/>
                    <group>
                        <group>
                            <field name="name"/>
                        </group>
                        <group>
                            <field name="user_id" widget="many2one_avatar_user"/>
                        </group>
                    </group>
                    <notebook>
                        <page name="description" string="Description">
                            <field name="description"/>
                        </page>
                    </notebook>
                </sheet>
                <chatter/>
            </form>
        </field>
    </record>
</odoo>
```

### Inherit View

```xml
<record id="view_model_form_inherit" model="ir.ui.view">
    <field name="name">model.form.inherit</field>
    <field name="model">model.name</field>
    <field name="inherit_id" ref="module.view_model_form"/>
    <field name="arch" type="xml">
        <xpath expr="//field[@name='name']" position="after">
            <field name="new_field"/>
        </xpath>
    </field>
</record>
```

## Security

### Access Rights CSV

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_project_user,project.user,model_project,group_project_user,1,1,1,0
access_project_manager,project.manager,model_project,group_project_manager,1,1,1,1
```

### Record Rules XML

```xml
<record model="ir.rule" id="project_comp_rule">
    <field name="name">Project: multi-company</field>
    <field name="model_id" ref="model_project_project"/>
    <field name="domain_force">[('company_id', 'in', company_ids + [False])]</field>
</record>
```

## Testing

### Test Class Structure

```python
from odoo import Command, fields
from odoo.tests import Form, users
from odoo.tests.common import TransactionCase
from odoo.exceptions import UserError

class TestProjectCommon(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super(TestProjectCommon, cls).setUpClass()
        cls.project = cls.env['project.project'].create({
            'name': 'Test Project',
        })

    def test_feature_description(self):
        """Test description explaining what is being tested."""
        # Arrange
        task = self.env['project.task'].create({
            'name': 'Test Task',
            'project_id': self.project.id,
        })
        # Act
        result = task.action_complete()
        # Assert
        self.assertEqual(task.state, '1_done')
        self.assertTrue(result)

    @users('project_user')
    def test_as_user(self):
        """Test running as specific user."""
        self.assertEqual(self.env.user.login, 'project_user')
```

### Test Patterns

```python
# Testing exceptions
with self.assertRaises(UserError):
    project.with_user(user).write({'name': 'New Name'})

# Using Form for UI testing
with Form(project) as form:
    form.name = 'Updated Name'
    form.is_favorite = True
self.assertTrue(project.is_favorite)

# Flushing and invalidating for access rights tests
project.flush_model()
project.invalidate_model()
```

## JavaScript/OWL

### Component Registration

```javascript
/** @odoo-module */

import { registry } from "@web/core/registry"
import { kanbanView } from "@web/views/kanban/kanban_view"
import { ProjectTaskKanbanModel } from "./project_task_kanban_model"

export const projectTaskKanbanView = {
  ...kanbanView,
  Model: ProjectTaskKanbanModel,
}

registry.category("views").add("project_task_kanban", projectTaskKanbanView)
```

### Field Widget

```javascript
import { registry } from "@web/core/registry"
import { booleanFavoriteField } from "@web/views/fields/boolean_favorite/boolean_favorite_field"

export const projectIsFavoriteField = {
  ...booleanFavoriteField,
  extractProps: (fieldsInfo, dynamicInfo) => {
    return {
      ...booleanFavoriteField.extractProps(fieldsInfo, dynamicInfo),
      readonly: Boolean(fieldsInfo.attrs.readonly),
    }
  },
}

registry.category("fields").add("project_is_favorite", projectIsFavoriteField)
```

## Common Anti-Patterns to Avoid

1. **Do NOT use type hints** - Odoo 18 codebase does not use Python type annotations
2. **Do NOT use f-strings in translations** - Use `_("%s value", var)` or `_("text %(key)s", key=val)`
3. **Do NOT add `# -*- coding: utf-8 -*-`** - Not needed in Odoo 18 (note: some legacy files still have it)
4. **Do NOT use `self.env.cr.execute()` for simple queries** - Use `_read_group()` instead
5. **Do NOT hardcode user IDs** - Use `self.env.uid` or `self.env.user`
6. **Do NOT bypass `super()` in CRUD methods** - Always call parent implementation
7. **Do NOT use `sudo()` without necessity** - Document why sudo is needed
8. **Do NOT skip `export_string_translation=False`** - Add to non-translatable computed fields
9. **Do NOT translate SQL constraint messages** - Use plain strings, not `_()`
10. **Do NOT orphan compute/inverse methods** - Always declare `compute=` and `inverse=` on the field definition
11. **Do NOT "fix" typos or improve existing code** - Match the original exactly unless style-related

## Running Tests

```bash
# Run all tests for a module
./odoo-bin -c odoo.conf -i project --test-enable

# Run specific test file
./odoo-bin -c odoo.conf -i project --test-enable --test-file=addons/project/tests/test_project_base.py

# Run specific test class/method (use pytest-style path)
./odoo-bin -c odoo.conf -i project --test-enable --test-file=addons/project/tests/test_project_base.py::TestProjectBase::test_feature
```
