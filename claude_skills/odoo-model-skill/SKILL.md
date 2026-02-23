---
name: odoo-model
description: Use this skill whenever creating, editing, or extending Odoo Python models — including new models, inherited models, fields, computed fields, constraints, CRUD overrides, business logic methods, and ORM queries. Trigger this skill any time the user asks to add a field, write a method, create a model file, extend an existing model, write a domain, fix an ORM query, or scaffold any Python code that will live in an Odoo module's `models/` directory. Also use for writing `__manifest__.py`, `__init__.py`, and `ir.sequence` definitions. When in doubt, use this skill — it's better to have the context than to miss it.
---

# Odoo Model Development

This skill helps you write correct, idiomatic, performant Odoo Python model code. Odoo has strong conventions — deviating from them causes subtle bugs, poor upgrade compatibility, and broken access rules.

---

## Model Anatomy

```python
from odoo import api, fields, models, _
from odoo.exceptions import UserError, ValidationError
import logging

_logger = logging.getLogger(__name__)


class MyModel(models.Model):
    # 1. Identity
    _name = 'my.module.thing'          # Required for new models
    _description = 'My Thing'          # Required, shown in UI
    _inherit = ['mail.thread',         # Add only when needed
                'mail.activity.mixin']
    _order = 'date desc, id desc'      # Default sort
    _rec_name = 'name'                 # Field used as display name
    _table = 'my_module_thing'         # Override only if needed

    # 2. SQL constraints (fast, DB-level)
    _sql_constraints = [
        ('name_uniq', 'UNIQUE(name, company_id)', 'Name must be unique per company.'),
    ]

    # 3. Fields
    # 4. @api.depends / computed fields
    # 5. @api.onchange
    # 6. @api.constrains
    # 7. CRUD overrides (create, write, unlink)
    # 8. Action / button methods
    # 9. Helper / private methods (prefix with _)
```

The attribute and method order above matters for readability and is the community standard. Follow it.

---

## Fields

### Type Reference

```python
# Scalar
name        = fields.Char(string='Name', required=True, index=True)
description = fields.Text(string='Description')
notes       = fields.Html(string='Notes')
amount      = fields.Float(string='Amount', digits='Product Price')
qty         = fields.Integer(string='Quantity', default=1)
price       = fields.Monetary(string='Price', currency_field='currency_id')
active      = fields.Boolean(default=True)
date        = fields.Date(string='Date', default=fields.Date.today)
datetime    = fields.Datetime(string='Date Time', default=fields.Datetime.now)

# Relational
partner_id  = fields.Many2one('res.partner', string='Partner',
                              ondelete='restrict', index=True)
tag_ids     = fields.Many2many('res.partner.category', string='Tags')
line_ids    = fields.One2many('my.module.line', 'order_id', string='Lines')

# Special
currency_id = fields.Many2one('res.currency', related='company_id.currency_id', store=True)
state       = fields.Selection([('draft','Draft'),('done','Done')],
                               default='draft', required=True, tracking=True, index=True)
company_id  = fields.Many2one('res.company', default=lambda self: self.env.company, index=True)
```

### Key Field Kwargs

| Kwarg | When to use |
|---|---|
| `required=True` | Must have a value |
| `index=True` | Filtered/sorted often — add for FK and state fields |
| `tracking=True` | Log changes in chatter (requires `mail.thread`) |
| `copy=False` | Don't duplicate on record copy (e.g., sequence numbers) |
| `store=True` | Store computed value in DB (enables search/group by) |
| `ondelete='restrict'` | Prevent deleting parent if children exist (safer than cascade) |
| `groups='base.group_system'` | Restrict field visibility to a security group |

---

## Computed Fields

```python
# Always use @api.depends — it controls when recomputation triggers
@api.depends('line_ids.price_subtotal', 'line_ids.discount')
def _compute_amount_total(self):
    for record in self:                        # Always loop over self
        record.amount_total = sum(
            record.line_ids.mapped('price_subtotal')
        )

# Computed + inverse (makes the field writable)
discount_pct = fields.Float(compute='_compute_discount', inverse='_set_discount', store=False)

@api.depends('discount_amount', 'amount_untaxed')
def _compute_discount(self):
    for rec in self:
        rec.discount_pct = (rec.discount_amount / rec.amount_untaxed * 100
                            if rec.amount_untaxed else 0.0)

def _set_discount(self):
    for rec in self:
        rec.discount_amount = rec.amount_untaxed * rec.discount_pct / 100
```

**Common pitfalls:**
- Forgetting `store=True` when you need to search/filter by the field
- Using `@api.depends('')` (empty string) as a lazy "always recompute" — this kills performance
- Not looping — always `for record in self`, never treat `self` as a singleton in compute methods

---

## Onchange

```python
@api.onchange('partner_id')
def _onchange_partner_id(self):
    if self.partner_id:
        self.payment_term_id = self.partner_id.property_payment_term_id
    else:
        self.payment_term_id = False
```

Onchange only fires in the UI form — never rely on it for data integrity. Put real business logic in compute fields or `write`/`create` overrides.

---

## Constraints

```python
# Python constraint — for multi-field or complex validation
@api.constrains('date_start', 'date_end')
def _check_dates(self):
    for rec in self:
        if rec.date_start and rec.date_end and rec.date_start > rec.date_end:
            raise ValidationError(_('Start date must be before end date.'))

# SQL constraint — always prefer for simple uniqueness / check constraints
_sql_constraints = [
    ('ref_company_uniq', 'UNIQUE(ref, company_id)', 'Reference must be unique per company.'),
    ('qty_positive',     'CHECK(qty >= 0)',          'Quantity cannot be negative.'),
]
```

---

## CRUD Overrides

```python
@api.model_create_multi           # Use this in Odoo 16+ (replaces @api.model for create)
def create(self, vals_list):
    for vals in vals_list:
        if vals.get('name', _('New')) == _('New'):
            vals['name'] = self.env['ir.sequence'].next_by_code('my.module.thing') or _('New')
    return super().create(vals_list)

def write(self, vals):
    # Guard against invalid state transitions
    if 'state' in vals and vals['state'] == 'done':
        for rec in self:
            if not rec.line_ids:
                raise UserError(_('Cannot complete "%s": no lines.') % rec.name)
    return super().write(vals)

def unlink(self):
    for rec in self:
        if rec.state not in ('draft', 'cancelled'):
            raise UserError(_('Only draft or cancelled records can be deleted.'))
    return super().unlink()

def copy(self, default=None):
    default = dict(default or {})
    default['name'] = _('%s (copy)') % self.name
    return super().copy(default)
```

Always call `super()` — skipping it breaks inheritance and can silently corrupt data.

---

## ORM Performance

```python
# BAD — triggers a query per record (N+1)
for order in orders:
    total += order.partner_id.credit_limit

# GOOD — Odoo prefetches relational fields when you access them on a recordset
limits = orders.mapped('partner_id.credit_limit')

# BAD — search inside a loop
for partner in partners:
    orders = self.env['sale.order'].search([('partner_id', '=', partner.id)])

# GOOD — single query with IN
orders = self.env['sale.order'].search([('partner_id', 'in', partners.ids)])

# Use search_read for read-only data (avoids ORM overhead)
rows = self.env['sale.order'].search_read(
    domain=[('state', '=', 'sale')],
    fields=['name', 'partner_id', 'amount_total'],
    limit=200,
)

# Use filtered/mapped/sorted instead of Python list comprehensions on recordsets
confirmed = orders.filtered(lambda o: o.state == 'sale')
names     = orders.mapped('name')
by_date   = orders.sorted('date_order', reverse=True)

# Bulk write (one UPDATE instead of N)
orders.write({'state': 'done'})   # Works on multi-record sets

# Use _compute_fields_value for efficient multi-field compute
```

---

## Context, Sudo, and Company

```python
# sudo() — bypass access rights (use sparingly, log the reason)
product = self.env['product.product'].sudo().browse(product_id)

# with_context() — pass context flags
order.with_context(no_recompute=True).write({'state': 'done'})

# with_company() — switch active company for multi-company records
record.with_company(self.env.company).create(vals)

# Accessing current user / company
current_user    = self.env.user
current_company = self.env.company
is_admin        = self.env.user.has_group('base.group_system')
```

---

## Common Patterns

### Sequence Numbers
```python
# In create():
vals['name'] = self.env['ir.sequence'].next_by_code('my.module.thing') or '/'
```

### Sending Email
```python
template = self.env.ref('my_module.email_template_confirmation')
template.send_mail(self.id, force_send=True)
```

### Posting to Chatter
```python
self.message_post(
    body=_('Status changed to <b>%s</b>.') % self.state,
    subtype_xmlid='mail.mt_note',
)
```

### Returning an Action
```python
def action_open_lines(self):
    self.ensure_one()
    return {
        'type': 'ir.actions.act_window',
        'name': _('Lines'),
        'res_model': 'my.module.line',
        'view_mode': 'list,form',
        'domain': [('order_id', '=', self.id)],
        'context': {'default_order_id': self.id},
    }
```

---

## Logging

```python
_logger = logging.getLogger(__name__)

_logger.info('Processing %d orders', len(orders))
_logger.warning('Skipping order %s: no partner', order.name)
_logger.error('Failed to send email for order %s', order.name, exc_info=True)
```

Never use `print()` in production code.

---

## Reference Files

- See `references/field-types.md` for the full field type reference with all kwargs
- See `references/orm-methods.md` for the full ORM method reference (search, read, write, etc.)
