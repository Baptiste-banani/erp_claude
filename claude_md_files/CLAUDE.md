# Odoo Development Best Practices

This file provides guidelines and best practices for developing in the Odoo framework. Follow these conventions to ensure maintainable, performant, and idiomatic Odoo code.

---

## Project Structure

An Odoo module must follow this directory layout:

```
my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
├── views/
│   └── my_model_views.xml
├── security/
│   ├── ir.model.access.csv
│   └── my_module_security.xml
├── data/
│   └── my_module_data.xml
├── wizards/
│   ├── __init__.py
│   └── my_wizard.py
├── controllers/
│   ├── __init__.py
│   └── main.py
├── static/
│   └── description/
│       └── icon.png
└── tests/
    ├── __init__.py
    └── test_my_model.py
```

---

## `__manifest__.py`

important: When creating the __manifes__.py file, always ask for the correct odoo_verion and place this in [odoo_version]. This can only be integers. Throw error when chars are entered.

```python
{
    'name': 'My Module',
    'version': '[odoo_version].0.1.0.0',  # Odoo version prefix is mandatory
    'summary': 'Short description of the module',
    'author': 'Your Name',
    'website': 'https://yourwebsite.com',
    'category': 'Uncategorized',
    'depends': ['base', 'mail'],
    'data': [
        'security/ir.model.access.csv',
        'security/my_module_security.xml',
        'data/my_module_data.xml',
        'views/my_model_views.xml',
    ],
    'installable': True,
    'application': False,
    'license': 'LGPL-3',
}
```

---

## Python / Model Best Practices

### General Rules

- Always inherit from `models.Model`, `models.TransientModel`, or `models.AbstractModel`.
- Use `_name` for new models and `_inherit` to extend existing ones.
- Never use `_inherit` and `_name` together unless creating a new model that copies another.
- Use `_description` — it is required and shown in the UI.

### Field Definitions

```python
from odoo import api, fields, models, _
from odoo.exceptions import UserError, ValidationError


class SaleOrder(models.Model):
    _name = 'my.module.order'
    _description = 'My Module Order'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'date_order desc, id desc'
    _rec_name = 'name'

    name = fields.Char(
        string='Order Reference',
        required=True,
        copy=False,
        readonly=True,
        default=lambda self: _('New'),
        index=True,
    )
    date_order = fields.Datetime(
        string='Order Date',
        required=True,
        default=fields.Datetime.now,
        tracking=True,
    )
    partner_id = fields.Many2one(
        comodel_name='res.partner',
        string='Customer',
        required=True,
        ondelete='restrict',
        index=True,
        tracking=True,
    )
    line_ids = fields.One2many(
        comodel_name='my.module.order.line',
        inverse_name='order_id',
        string='Order Lines',
        copy=True,
    )
    amount_total = fields.Monetary(
        string='Total Amount',
        compute='_compute_amount_total',
        store=True,
    )
    currency_id = fields.Many2one(
        comodel_name='res.currency',
        related='partner_id.currency_id',
        store=True,
    )
    state = fields.Selection(
        selection=[
            ('draft', 'Draft'),
            ('confirmed', 'Confirmed'),
            ('done', 'Done'),
            ('cancelled', 'Cancelled'),
        ],
        string='Status',
        default='draft',
        required=True,
        tracking=True,
        index=True,
    )
    notes = fields.Html(string='Internal Notes')
    active = fields.Boolean(default=True)
```

### Computed Fields

```python
    @api.depends('line_ids.price_subtotal')
    def _compute_amount_total(self):
        for order in self:
            order.amount_total = sum(order.line_ids.mapped('price_subtotal'))
```

- Always use `@api.depends` with computed fields.
- Use `store=True` when the field needs to be searched or grouped.
- Loop over `self` — never assume a singleton unless using `ensure_one()`.

### Onchange Methods

```python
    @api.onchange('partner_id')
    def _onchange_partner_id(self):
        if self.partner_id:
            self.currency_id = self.partner_id.currency_id
        else:
            self.currency_id = False
```

- Onchange only runs in the UI. Use `@api.depends` for business logic that must be reliable.

### Constraints

```python
    @api.constrains('date_order', 'line_ids')
    def _check_order_validity(self):
        for order in self:
            if order.date_order and order.date_order > fields.Datetime.now():
                raise ValidationError(_('Order date cannot be in the future.'))

    _sql_constraints = [
        ('name_uniq', 'UNIQUE(name)', 'Order reference must be unique.'),
        ('amount_positive', 'CHECK(amount_total >= 0)', 'Total amount must be positive.'),
    ]
```

- Use `_sql_constraints` for simple DB-level constraints (faster).
- Use `@api.constrains` for complex Python-level validation.

### CRUD Overrides

```python
    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if vals.get('name', _('New')) == _('New'):
                vals['name'] = self.env['ir.sequence'].next_by_code('my.module.order') or _('New')
        return super().create(vals_list)

    def write(self, vals):
        if 'state' in vals and vals['state'] == 'done':
            for record in self:
                if not record.line_ids:
                    raise UserError(_('Cannot confirm an order with no lines.'))
        return super().write(vals)

    def unlink(self):
        for record in self:
            if record.state not in ('draft', 'cancelled'):
                raise UserError(_('Only draft or cancelled orders can be deleted.'))
        return super().unlink()
```

- Use `@api.model_create_multi` instead of `@api.model` for `create` in Odoo 16+.
- Always call `super()` unless explicitly overriding the entire behavior.

### Business Logic Methods

```python
    def action_confirm(self):
        self.ensure_one()
        if self.state != 'draft':
            raise UserError(_('Only draft orders can be confirmed.'))
        self.write({'state': 'confirmed'})
        self.message_post(body=_('Order confirmed.'))

    def action_cancel(self):
        for order in self:
            if order.state == 'done':
                raise UserError(_('Done orders cannot be cancelled.'))
        self.write({'state': 'cancelled'})
```

- Use `ensure_one()` when the method is intended for a single record.
- Use `self.write({...})` instead of direct attribute assignment for batch updates.

### ORM Performance

```python
    # BAD — N+1 query
    for order in orders:
        print(order.partner_id.name)

    # GOOD — prefetch with mapped
    names = orders.mapped('partner_id.name')

    # BAD — loop with search inside
    for partner in partners:
        orders = self.env['my.module.order'].search([('partner_id', '=', partner.id)])

    # GOOD — search outside loop with IN clause
    orders = self.env['my.module.order'].search([('partner_id', 'in', partners.ids)])

    # Use read() for large datasets instead of browsing
    data = self.env['my.module.order'].search_read(
        domain=[('state', '=', 'confirmed')],
        fields=['name', 'partner_id', 'amount_total'],
    )
```

- Prefer `mapped()`, `filtered()`, and `sorted()` over Python list comprehensions on recordsets.
- Use `with_context()` and `sudo()` sparingly and intentionally.
- Use `_check_access_rights` or `check_access_rule` rather than bypassing with `sudo()` everywhere.

### Translations

```python
from odoo import _

# Always wrap user-visible strings with _()
raise UserError(_('This action is not allowed.'))
self.message_post(body=_('Record %s was updated.') % self.name)
```

---

## XML Best Practices

### Naming Conventions

All XML `id` attributes must be unique within a module and should follow the pattern:

```
<module_name>.<type>_<model>_<descriptor>
```

Examples:
- `my_module.view_my_order_form`
- `my_module.action_my_order`
- `my_module.menu_my_module_root`
- `my_module.sequence_my_order`
- `my_module.group_my_module_user`

### Form Views

```xml
<record id="view_my_order_form" model="ir.ui.view">
    <field name="name">my.module.order.form</field>
    <field name="model">my.module.order</field>
    <field name="arch" type="xml">
        <form string="Order">
            <header>
                <button name="action_confirm"
                        string="Confirm"
                        type="object"
                        class="oe_highlight"
                        invisible="state != 'draft'"/>
                <button name="action_cancel"
                        string="Cancel"
                        type="object"
                        invisible="state in ('done', 'cancelled')"/>
                <field name="state"
                       widget="statusbar"
                       statusbar_visible="draft,confirmed,done"/>
            </header>
            <sheet>
                <div class="oe_title">
                    <h1>
                        <field name="name" readonly="1"/>
                    </h1>
                </div>
                <group>
                    <group>
                        <field name="partner_id"
                               options="{'no_create': True}"
                               readonly="state != 'draft'"/>
                        <field name="date_order"
                               readonly="state != 'draft'"/>
                    </group>
                    <group>
                        <field name="currency_id" invisible="1"/>
                        <field name="amount_total"/>
                    </group>
                </group>
                <notebook>
                    <page string="Order Lines" name="order_lines">
                        <field name="line_ids"
                               widget="one2many_list"
                               readonly="state != 'draft'">
                            <tree editable="bottom">
                                <field name="product_id"/>
                                <field name="quantity"/>
                                <field name="price_unit"/>
                                <field name="price_subtotal"/>
                            </tree>
                        </field>
                    </page>
                    <page string="Notes" name="notes">
                        <field name="notes" placeholder="Add internal notes here..."/>
                    </page>
                </notebook>
            </sheet>
            <div class="oe_chatter">
                <field name="message_follower_ids"/>
                <field name="activity_ids"/>
                <field name="message_ids"/>
            </div>
        </form>
    </field>
</record>
```

### List (Tree) Views

```xml
<record id="view_my_order_list" model="ir.ui.view">
    <field name="name">my.module.order.list</field>
    <field name="model">my.module.order</field>
    <field name="arch" type="xml">
        <tree string="Orders" decoration-muted="state == 'cancelled'" decoration-success="state == 'done'">
            <field name="name"/>
            <field name="partner_id"/>
            <field name="date_order"/>
            <field name="amount_total" sum="Total"/>
            <field name="state" widget="badge"
                   decoration-info="state == 'draft'"
                   decoration-success="state == 'done'"
                   decoration-danger="state == 'cancelled'"/>
        </tree>
    </field>
</record>
```

### Search Views

```xml
<record id="view_my_order_search" model="ir.ui.view">
    <field name="name">my.module.order.search</field>
    <field name="model">my.module.order</field>
    <field name="arch" type="xml">
        <search string="Search Orders">
            <field name="name" string="Order"/>
            <field name="partner_id"/>
            <filter string="My Orders"
                    name="my_orders"
                    domain="[('partner_id.user_id', '=', uid)]"/>
            <filter string="Draft"
                    name="draft"
                    domain="[('state', '=', 'draft')]"/>
            <separator/>
            <filter string="This Month"
                    name="this_month"
                    domain="[('date_order', '&gt;=', (context_today() + relativedelta(day=1)).strftime('%Y-%m-%d'))]"/>
            <group expand="0" string="Group By">
                <filter string="Customer" name="group_partner" context="{'group_by': 'partner_id'}"/>
                <filter string="Status" name="group_state" context="{'group_by': 'state'}"/>
                <filter string="Month" name="group_month" context="{'group_by': 'date_order:month'}"/>
            </group>
        </search>
    </field>
</record>
```

### Actions and Menus

```xml
<record id="action_my_order" model="ir.actions.act_window">
    <field name="name">Orders</field>
    <field name="res_model">my.module.order</field>
    <field name="view_mode">list,form</field>
    <field name="search_view_id" ref="view_my_order_search"/>
    <field name="context">{'search_default_draft': 1}</field>
    <field name="help" type="html">
        <p class="o_view_nocontent_smiling_face">
            Create your first order!
        </p>
    </field>
</record>

<menuitem id="menu_my_module_root"
          name="My Module"
          sequence="10"
          web_icon="my_module,static/description/icon.png"/>

<menuitem id="menu_my_module_orders"
          name="Orders"
          parent="menu_my_module_root"
          action="action_my_order"
          sequence="10"/>
```

### Inheriting Views

important: When inheriting from a standard view, ask for the location of the view so you can find the correct view to inherit from.

```xml
<!-- Extend an existing view using xpath -->
<record id="view_sale_order_form_inherit_my_module" model="ir.ui.view">
    <field name="name">sale.order.form.inherit.my_module</field>
    <field name="model">sale.order</field>
    <field name="inherit_id" ref="sale.view_order_form"/>
    <field name="arch" type="xml">
        <!-- Append after an existing field -->
        <xpath expr="//field[@name='partner_id']" position="after">
            <field name="my_custom_field"/>
        </xpath>

        <!-- Replace an element -->
        <xpath expr="//button[@name='action_confirm']" position="replace">
            <button name="action_confirm_custom"
                    string="Confirm Custom"
                    type="object"
                    class="oe_highlight"/>
        </xpath>

        <!-- Add attributes to an existing element -->
        <xpath expr="//field[@name='note']" position="attributes">
            <attribute name="invisible">state != 'draft'</attribute>
        </xpath>
    </field>
</record>
```

- Prefer `position="after"` or `position="before"` over `position="replace"` to avoid breaking other extensions.
- Use `position="attributes"` to add/modify attributes without replacing the node.

### Security

```xml
<!-- security/my_module_security.xml -->
<odoo>
    <data noupdate="1">
        <record id="module_category_my_module" model="ir.module.category">
            <field name="name">My Module</field>
            <field name="sequence">10</field>
        </record>

        <record id="group_my_module_user" model="res.groups">
            <field name="name">User</field>
            <field name="category_id" ref="module_category_my_module"/>
            <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
        </record>

        <record id="group_my_module_manager" model="res.groups">
            <field name="name">Manager</field>
            <field name="category_id" ref="module_category_my_module"/>
            <field name="implied_ids" eval="[(4, ref('group_my_module_user'))]"/>
        </record>
    </data>
</odoo>
```

```csv
# security/ir.model.access.csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_order_user,my.module.order user,model_my_module_order,group_my_module_user,1,1,1,0
access_my_order_manager,my.module.order manager,model_my_module_order,group_my_module_manager,1,1,1,1
```

### Data Files

```xml
<!-- data/my_module_data.xml -->
<odoo>
    <data noupdate="1">  <!-- noupdate="1" for data that should not be reset on module update -->
        <record id="sequence_my_order" model="ir.sequence">
            <field name="name">My Module Order</field>
            <field name="code">my.module.order</field>
            <field name="prefix">ORD/%(year)s/</field>
            <field name="padding">5</field>
            <field name="company_id" eval="False"/>
        </record>
    </data>
</odoo>
```

---

## Controller Best Practices

```python
from odoo import http
from odoo.http import request


class MyModuleController(http.Controller):

    @http.route('/my_module/orders', type='http', auth='user', website=True)
    def list_orders(self, **kwargs):
        orders = request.env['my.module.order'].search([])
        return request.render('my_module.template_order_list', {'orders': orders})

    @http.route('/my_module/api/orders', type='json', auth='user', methods=['POST'])
    def get_orders_json(self, **kwargs):
        orders = request.env['my.module.order'].search_read(
            domain=[('state', '=', 'confirmed')],
            fields=['name', 'partner_id', 'amount_total'],
        )
        return {'orders': orders}
```

---

## Testing

```python
from odoo.tests.common import TransactionCase
from odoo.exceptions import UserError


class TestMyOrder(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Test Partner'})

    def test_order_confirmation(self):
        order = self.env['my.module.order'].create({
            'partner_id': self.partner.id,
        })
        self.assertEqual(order.state, 'draft')
        order.action_confirm()
        self.assertEqual(order.state, 'confirmed')

    def test_order_confirmation_no_lines_raises(self):
        order = self.env['my.module.order'].create({
            'partner_id': self.partner.id,
        })
        with self.assertRaises(UserError):
            order.action_confirm()
```

Run tests with:
```bash
python odoo-bin -d mydb --test-enable -i my_module --stop-after-init
```

---

## Common Pitfalls to Avoid

- **Never use `browse(id)` when you can use `search` or pass IDs directly** — it skips access checks.
- **Never commit inside a method** — Odoo manages transactions; use `cr.savepoint()` for partial rollbacks.
- **Avoid `env.ref()` in loops** — resolve refs outside loops.
- **Don't use `request.env.user` in background jobs** — use `self.env.user` or pass the user explicitly.
- **Don't override `fields_view_get`** — use `ir.ui.view` inheritance instead.
- **Don't hardcode IDs** — use XML IDs with `env.ref()` or `ref()` in XML.
- **Use `_logger` for logging, not `print()`**:

```python
import logging
_logger = logging.getLogger(__name__)

_logger.info('Processing order %s', self.name)
_logger.warning('Unexpected state: %s', self.state)
```

---

## Code Style

- Follow [PEP 8](https://peps.python.org/pep-0008/).
- Use 4-space indentation (no tabs).
- Maximum line length: 120 characters.
- Order model attributes: `_name`, `_description`, `_inherit`, `_inherits`, `_table`, `_order`, `_rec_name`, `_sql_constraints`, then fields, then `@api.depends`, `@api.onchange`, `@api.constrains`, then CRUD overrides, then business methods.
- Group related fields together with blank lines and comments where helpful.
- Always use keyword arguments for field definitions for clarity.
