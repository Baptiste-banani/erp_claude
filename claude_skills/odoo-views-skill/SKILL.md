---
name: odoo-views
description: Use this skill whenever writing or editing Odoo XML — including views (form, list/tree, search, kanban, pivot, graph, calendar), actions, menus, security groups, access rights (ir.model.access.csv), record rules, data files, email templates, and report templates (QWeb). Trigger this skill any time the user is working in an Odoo module's `views/`, `security/`, `data/`, or `report/` directory, or any time they ask to add a button, field, filter, menu item, access rule, or modify how something looks in the Odoo UI. Also use when inheriting/extending existing views with xpath. When in doubt, use this skill.
---

# Odoo Views & XML Development

This skill covers all Odoo XML work: views, security, data, menus, and actions. Odoo XML has strict conventions — getting them wrong causes silent failures, view inheritance conflicts, or security holes.

---

## XML File Structure

Every Odoo XML file must be wrapped in `<odoo>`. Use `noupdate="1"` on `<data>` only for records that shouldn't be reset on module update (e.g., sequences, security groups, demo data).

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data>
        <!-- Views, actions, menus go here (will be reset on --update) -->
    </data>
</odoo>

<!-- For seed data that should NOT be overwritten on update: -->
<odoo>
    <data noupdate="1">
        <!-- Sequences, email templates, default config, etc. -->
    </data>
</odoo>
```

---

## XML ID Naming Convention

XML IDs must be unique within a module. Follow this pattern:

```
<module_name>.<type>_<model_shortname>_<descriptor>
```

| Type prefix | Used for |
|---|---|
| `view_` | `ir.ui.view` records |
| `action_` | `ir.actions.*` records |
| `menu_` | `ir.ui.menu` records |
| `group_` | `res.groups` records |
| `rule_` | `ir.rule` records |
| `sequence_` | `ir.sequence` records |
| `template_` | QWeb templates / email templates |

Examples:
- `my_module.view_sale_order_form_inherit`
- `my_module.action_my_order`
- `my_module.menu_my_module_root`
- `my_module.group_my_module_manager`

---

## Form Views

```xml
<record id="view_my_order_form" model="ir.ui.view">
    <field name="name">my.order.form</field>
    <field name="model">my.module.order</field>
    <field name="arch" type="xml">
        <form string="Order">
            <header>
                <button name="action_confirm" string="Confirm" type="object"
                        class="oe_highlight" invisible="state != 'draft'"/>
                <button name="action_cancel"  string="Cancel"  type="object"
                        invisible="state in ('done', 'cancelled')"/>
                <field name="state" widget="statusbar"
                       statusbar_visible="draft,confirmed,done"/>
            </header>
            <sheet>
                <div class="oe_button_box" name="button_box">
                    <!-- Smart buttons go here -->
                    <button name="action_view_invoices" type="object"
                            class="oe_stat_button" icon="fa-pencil-square-o"
                            invisible="invoice_count == 0">
                        <field name="invoice_count" widget="statinfo" string="Invoices"/>
                    </button>
                </div>
                <div class="oe_title">
                    <h1><field name="name" readonly="1"/></h1>
                </div>
                <group>
                    <group>
                        <field name="partner_id" options="{'no_create': True}"
                               readonly="state != 'draft'"/>
                        <field name="date_order"  readonly="state != 'draft'"/>
                    </group>
                    <group>
                        <field name="currency_id" invisible="1"/>
                        <field name="amount_total"/>
                    </group>
                </group>
                <notebook>
                    <page string="Lines" name="order_lines">
                        <field name="line_ids" readonly="state != 'draft'">
                            <list editable="bottom">
                                <field name="product_id"/>
                                <field name="qty"/>
                                <field name="price_unit"/>
                                <field name="price_subtotal" optional="show"/>
                            </list>
                        </field>
                    </page>
                    <page string="Notes" name="notes">
                        <field name="notes" placeholder="Internal notes..."/>
                    </page>
                </notebook>
            </sheet>
            <chatter/>   <!-- Odoo 17+ shorthand -->
            <!-- Odoo 16 and earlier: -->
            <!--
            <div class="oe_chatter">
                <field name="message_follower_ids"/>
                <field name="activity_ids"/>
                <field name="message_ids"/>
            </div>
            -->
        </form>
    </field>
</record>
```

### `invisible`, `readonly`, `required` Attributes

In Odoo 17+ use Python-like inline expressions directly:

```xml
<field name="my_field" invisible="state != 'draft'"/>
<field name="my_field" readonly="state in ('done', 'cancelled')"/>
<button invisible="amount_total == 0 or state != 'draft'"/>
```

In Odoo 16 and earlier, use `attrs=`:
```xml
<field name="my_field" attrs="{'invisible': [('state', '!=', 'draft')]}"/>
```

---

## List (Tree) Views

```xml
<record id="view_my_order_list" model="ir.ui.view">
    <field name="name">my.order.list</field>
    <field name="model">my.module.order</field>
    <field name="arch" type="xml">
        <list string="Orders"
              decoration-muted="state == 'cancelled'"
              decoration-success="state == 'done'"
              decoration-info="state == 'draft'">
            <field name="name"/>
            <field name="partner_id"/>
            <field name="date_order"/>
            <field name="amount_total" sum="Total" optional="show"/>
            <field name="state" widget="badge"
                   decoration-success="state == 'done'"
                   decoration-danger="state == 'cancelled'"
                   decoration-info="state == 'draft'"/>
        </list>
    </field>
</record>
```

Useful list attributes: `editable="bottom"` (inline editing), `multi_edit="1"` (bulk edits), `optional="show|hide"` (user-toggleable column).

---

## Search Views

```xml
<record id="view_my_order_search" model="ir.ui.view">
    <field name="name">my.order.search</field>
    <field name="model">my.module.order</field>
    <field name="arch" type="xml">
        <search string="Search Orders">
            <!-- Searchable fields -->
            <field name="name" string="Reference"/>
            <field name="partner_id" operator="child_of"/>

            <!-- Predefined filters -->
            <filter string="My Orders" name="my_orders"
                    domain="[('user_id', '=', uid)]"/>
            <filter string="Draft" name="draft"
                    domain="[('state', '=', 'draft')]"/>
            <separator/>
            <filter string="This Month" name="this_month"
                    domain="[('date_order', '&gt;=', (context_today() + relativedelta(day=1)).strftime('%Y-%m-%d'))]"/>

            <!-- Group By options -->
            <group expand="0" string="Group By">
                <filter string="Customer" name="group_partner"
                        context="{'group_by': 'partner_id'}"/>
                <filter string="Status"   name="group_state"
                        context="{'group_by': 'state'}"/>
                <filter string="Month"    name="group_month"
                        context="{'group_by': 'date_order:month'}"/>
            </group>
        </search>
    </field>
</record>
```

---

## Kanban Views

```xml
<record id="view_my_order_kanban" model="ir.ui.view">
    <field name="name">my.order.kanban</field>
    <field name="model">my.module.order</field>
    <field name="arch" type="xml">
        <kanban default_group_by="state" class="o_kanban_small_column">
            <field name="name"/>
            <field name="partner_id"/>
            <field name="amount_total"/>
            <field name="state"/>
            <field name="color"/>
            <templates>
                <t t-name="card">
                    <div t-attf-class="oe_kanban_color_#{record.color.raw_value}">
                        <div class="oe_kanban_content">
                            <strong><field name="name"/></strong>
                            <div><field name="partner_id"/></div>
                            <div class="text-muted"><field name="amount_total"/> </div>
                        </div>
                    </div>
                </t>
            </templates>
        </kanban>
    </field>
</record>
```

---

## Actions

```xml
<!-- Window action — opens a model in a view -->
<record id="action_my_order" model="ir.actions.act_window">
    <field name="name">Orders</field>
    <field name="res_model">my.module.order</field>
    <field name="view_mode">list,form,kanban</field>
    <field name="search_view_id" ref="view_my_order_search"/>
    <field name="context">{'search_default_my_orders': 1}</field>
    <field name="domain">[]</field>
    <field name="help" type="html">
        <p class="o_view_nocontent_smiling_face">Create your first order!</p>
    </field>
</record>

<!-- Server action — runs Python code or calls a method -->
<record id="action_send_reminder" model="ir.actions.server">
    <field name="name">Send Reminder</field>
    <field name="model_id" ref="model_my_module_order"/>
    <field name="binding_model_id" ref="model_my_module_order"/>  <!-- Shows in Action menu -->
    <field name="state">code</field>
    <field name="code">records.action_send_reminder()</field>
</record>
```

---

## Menus

```xml
<!-- Root menu (top-level app icon) -->
<menuitem id="menu_my_module_root"
          name="My Module"
          sequence="50"
          web_icon="my_module,static/description/icon.png"/>

<!-- Category menu -->
<menuitem id="menu_my_module_orders"
          name="Orders"
          parent="menu_my_module_root"
          sequence="10"/>

<!-- Leaf menu — must have action -->
<menuitem id="menu_my_order_list"
          name="All Orders"
          parent="menu_my_module_orders"
          action="action_my_order"
          sequence="10"/>
```

---

## View Inheritance (xpath)

The safest extension approach — always prefer `after/before` over `replace`.

```xml
<record id="view_sale_order_form_inherit_my_module" model="ir.ui.view">
    <field name="name">sale.order.form.inherit.my_module</field>
    <field name="model">sale.order</field>
    <field name="inherit_id" ref="sale.view_order_form"/>
    <field name="arch" type="xml">

        <!-- Add a field after an existing one -->
        <xpath expr="//field[@name='partner_id']" position="after">
            <field name="my_custom_field"/>
        </xpath>

        <!-- Add to an existing group -->
        <xpath expr="//group[@name='sale_header_left']" position="inside">
            <field name="my_extra_field"/>
        </xpath>

        <!-- Modify attributes without replacing the element -->
        <xpath expr="//field[@name='note']" position="attributes">
            <attribute name="invisible">state != 'draft'</attribute>
        </xpath>

        <!-- Add a button to the header -->
        <xpath expr="//button[@name='action_confirm']" position="before">
            <button name="action_my_custom_step" string="Custom Step"
                    type="object" class="oe_highlight"
                    invisible="state != 'draft'"/>
        </xpath>

        <!-- Replace a field widget -->
        <xpath expr="//field[@name='payment_term_id']" position="replace">
            <field name="payment_term_id" widget="selection"/>
        </xpath>

    </field>
</record>
```

Avoid `position="replace"` whenever possible — it breaks other modules that try to extend the same element.

---

## Security

### `security/ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_order_user,my.order user,model_my_module_order,my_module.group_my_module_user,1,1,1,0
access_my_order_manager,my.order manager,model_my_module_order,my_module.group_my_module_manager,1,1,1,1
access_my_order_public,my.order public,model_my_module_order,,1,0,0,0
```

Every model must have at least one access rule or it's invisible to all non-admin users.

### Security Groups

```xml
<!-- security/my_module_security.xml -->
<odoo>
    <data noupdate="0">
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
            <field name="users" eval="[(4, ref('base.user_root')), (4, ref('base.user_admin'))]"/>
        </record>
    </data>
</odoo>
```

### Record Rules

```xml
<record id="rule_my_order_personal" model="ir.rule">
    <field name="name">My Order: personal</field>
    <field name="model_id" ref="model_my_module_order"/>
    <field name="domain_force">[('user_id', '=', user.id)]</field>
    <field name="groups" eval="[(4, ref('my_module.group_my_module_user'))]"/>
    <field name="perm_read"   eval="True"/>
    <field name="perm_write"  eval="True"/>
    <field name="perm_create" eval="True"/>
    <field name="perm_unlink" eval="False"/>
</record>
```

---

## Data Files

```xml
<!-- data/my_module_data.xml -->
<odoo>
    <data noupdate="1">
        <record id="sequence_my_order" model="ir.sequence">
            <field name="name">My Module Order</field>
            <field name="code">my.module.order</field>
            <field name="prefix">ORD/%(year)s/%(month)s/</field>
            <field name="padding">5</field>
            <field name="company_id" eval="False"/>  <!-- False = shared across companies -->
        </record>
    </data>
</odoo>
```

---

## QWeb Reports

```xml
<!-- report/my_order_report.xml -->
<odoo>
    <record id="action_report_my_order" model="ir.actions.report">
        <field name="name">Order Report</field>
        <field name="model">my.module.order</field>
        <field name="report_type">qweb-pdf</field>
        <field name="report_name">my_module.report_my_order_document</field>
        <field name="report_file">my_module.report_my_order_document</field>
        <field name="binding_model_id" ref="model_my_module_order"/>
        <field name="binding_type">report</field>
    </record>

    <template id="report_my_order_document">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="doc">
                <t t-call="web.external_layout">
                    <div class="page">
                        <h2><t t-esc="doc.name"/></h2>
                        <table class="table table-sm">
                            <thead>
                                <tr>
                                    <th>Product</th>
                                    <th class="text-right">Qty</th>
                                    <th class="text-right">Price</th>
                                </tr>
                            </thead>
                            <tbody>
                                <tr t-foreach="doc.line_ids" t-as="line">
                                    <td><t t-esc="line.product_id.name"/></td>
                                    <td class="text-right"><t t-esc="line.qty"/></td>
                                    <td class="text-right"><t t-esc="line.price_unit"/></td>
                                </tr>
                            </tbody>
                        </table>
                    </div>
                </t>
            </t>
        </t>
    </template>
</odoo>
```

---

## Common Mistakes to Avoid

- **Missing `noupdate="1"`** on sequences and configuration data — they'll be reset every upgrade.
- **Using `position="replace"`** on inherited views — breaks other modules extending the same node.
- **Referencing an XML ID before it's declared** — order your `data:` list in `__manifest__.py` so security files come before views.
- **Forgetting the model in `ir.model.access.csv`** — the `model_id:id` must match the model's XML ID (auto-generated as `model_<model_name_with_underscores>`).
- **Using `<tree>` instead of `<list>`** — in Odoo 17+ the tag is `<list>`, though `<tree>` still works for now.
- **Domain syntax in XML** — use `&amp;` for `&`, `&lt;` for `<`, `&gt;` for `>` inside XML attribute values. Or wrap in `<![CDATA[...]]>`.
