# Especificación Técnica: Sistema de Control y Validación de Retenciones, Percepciones y Archivos TXT

**Versión:** 1.0 
**Fecha:** 4 de mayo de 2026  
**Autor:** Equipo Técnico ADHOC  
**Producto:** Liquidación de Impuestos / Localizaciones Argentina

---

## 1. Resumen Ejecutivo

### 1.1 Objetivo
Implementar un sistema de control preventivo que valide datos obligatorios, explique aplicación/no aplicación de retenciones y percepciones, identifique inconsistencias y valide archivos TXT antes de su presentación a organismos fiscales.

### 1.2 Alcance Técnico
- **Módulo nuevo:** `l10n_ar_tax_settlement_validation` (Enterprise)
- **Módulos afectados:** `l10n_ar_account_tax_settlement`, `l10n_ar_account_reports`, módulos provinciales de TXT
- **Versión Odoo:** 18.0

### 1.3 Métricas de Éxito
- Reducción ≥25% de tickets relacionados en 60 días post-lanzamiento
- Tiempo de validación inicial <3 segundos para periodos mensuales
- 100% de errores críticos detectados antes de la exportación

---

## 2. Arquitectura Propuesta

### 2.1 Diagrama de Componentes

```
┌─────────────────────────────────────────────────────────────┐
│                     account.tax.report                      │
│                   (Reporte de Liquidación)                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│         l10n_ar.tax.settlement.validation.mixin             │
│  • pre_export_validation()                                  │
│  • get_validation_summary()                                 │
│  • explain_retention_application()                          │
└──────────────┬──────────────────────────────────────────────┘
               │
               ├────────────────┬────────────────┬─────────────┐
               ▼                ▼                ▼             ▼
    ┌─────────────────┐ ┌──────────────┐ ┌──────────────┐ ┌─────────────┐
    │ Data Validator  │ │  Rule Engine │ │  Explainer   │ │ TXT Checker │
    └─────────────────┘ └──────────────┘ └──────────────┘ └─────────────┘
               │                │                │             │
               └────────────────┴────────────────┴─────────────┘
                                 │
                                 ▼
                ┌────────────────────────────────────┐
                │  l10n_ar.tax.validation.wizard     │
                │  (Wizard de Validación Interactivo)│
                └────────────────────────────────────┘
```

### 2.2 Componentes Principales

#### 2.2.1 Modelo: `l10n_ar.tax.validation.result`
**Propósito:** Almacenar resultados de validación para auditoría y debugging

**Campos:**
```python
_name = 'l10n_ar.tax.validation.result'
_description = 'Tax Settlement Validation Result'
_order = 'create_date desc'

# Relaciones
report_id = fields.Many2one('account.tax.report', required=True, ondelete='cascade')
company_id = fields.Many2one('res.company', required=True)
period_start = fields.Date(required=True)
period_end = fields.Date(required=True)

# Estado de validación
validation_state = fields.Selection([
    ('valid', 'Válido'),
    ('warning', 'Advertencias'),
    ('error', 'Errores críticos'),
    ('blocked', 'Bloqueado'),
], default='valid', index=True)

# Resultados
error_count = fields.Integer(compute='_compute_counts', store=True)
warning_count = fields.Integer(compute='_compute_counts', store=True)
info_count = fields.Integer(compute='_compute_counts', store=True)

# Detalles
line_ids = fields.One2many('l10n_ar.tax.validation.line', 'validation_id')

# Contexto
export_format = fields.Selection([
    ('sire', 'SIRE'),
    ('sicore', 'SICORE'),
    ('arba', 'ARBA'),
    ('caba', 'CABA'),
    ('santa_fe', 'Santa Fe'),
    ('tucuman', 'Tucumán'),
    ('mendoza', 'Mendoza'),
    # ... otras jurisdicciones
], string='Formato de exportación')

# Timestamp
validation_time = fields.Float(string='Tiempo de validación (segundos)')
```

#### 2.2.2 Modelo: `l10n_ar.tax.validation.line`
**Propósito:** Líneas individuales de validación (errores, advertencias, info)

**Campos:**
```python
_name = 'l10n_ar.tax.validation.line'
_description = 'Tax Validation Line'
_order = 'severity desc, sequence'

validation_id = fields.Many2one('l10n_ar.tax.validation.result', required=True, ondelete='cascade')

# Tipo y severidad
severity = fields.Selection([
    ('error', 'Error crítico'),
    ('warning', 'Advertencia'),
    ('info', 'Información'),
], required=True, index=True)

category = fields.Selection([
    ('missing_data', 'Datos faltantes'),
    ('invalid_format', 'Formato inválido'),
    ('padron', 'Padrón de contribuyentes'),
    ('tax_config', 'Configuración de impuestos'),
    ('document', 'Comprobante'),
    ('computation', 'Cálculo'),
    ('regulatory', 'Cumplimiento normativo'),
], required=True)

# Mensaje
code = fields.Char(required=True, index=True)  # ej: 'SIRE_001', 'ARBA_VAT_MISSING'
title = fields.Char(required=True)
message = fields.Text(required=True)
technical_detail = fields.Text()  # Para debugging

# Referencias
res_model = fields.Char(index=True)  # Modelo relacionado
res_id = fields.Integer(index=True)  # ID del registro
document_ref = fields.Char()  # Referencia al documento (ej: factura)

# Acción sugerida
action_type = fields.Selection([
    ('fix_required', 'Corrección requerida'),
    ('review', 'Revisar'),
    ('ignore', 'Puede ignorarse'),
    ('automatic', 'Puede corregirse automáticamente'),
], default='review')

action_button = fields.Char()  # Nombre del botón de acción
action_method = fields.Char()  # Método a ejecutar

sequence = fields.Integer(default=10)
```

#### 2.2.3 Mixin: `l10n_ar.tax.settlement.validation.mixin`
**Propósito:** Lógica reutilizable de validación para reportes

```python
class L10nArTaxSettlementValidationMixin(models.AbstractModel):
    _name = 'l10n_ar.tax.settlement.validation.mixin'
    _description = 'Tax Settlement Validation Mixin'

    def validate_before_export(self, options, export_format):
        """
        Punto de entrada principal para validación antes de exportar.
        
        Args:
            options (dict): Opciones del reporte (periodo, filtros, etc.)
            export_format (str): Formato de exportación (sire, arba, etc.)
            
        Returns:
            l10n_ar.tax.validation.result: Resultado de validación
        """
        self.ensure_one()
        start_time = time.time()
        
        # Crear resultado de validación
        validation = self.env['l10n_ar.tax.validation.result'].create({
            'report_id': self.id,
            'company_id': self.env.company.id,
            'period_start': options['date']['date_from'],
            'period_end': options['date']['date_to'],
            'export_format': export_format,
        })
        
        # Ejecutar validadores
        validators = self._get_validators(export_format)
        lines = []
        
        for validator in validators:
            try:
                result = validator(options, export_format)
                if result:
                    lines.extend(result)
            except Exception as e:
                lines.append(self._create_error_line(
                    'SYSTEM_ERROR',
                    'Error en validación',
                    str(e),
                    'error',
                    'regulatory'
                ))
        
        # Guardar líneas
        validation.line_ids = [(0, 0, line) for line in lines]
        validation.validation_time = time.time() - start_time
        
        return validation

    def _get_validators(self, export_format):
        """Retorna lista de métodos validadores según formato"""
        base_validators = [
            self._validate_partner_data,
            self._validate_tax_configuration,
            self._validate_document_integrity,
            self._validate_amounts,
        ]
        
        # Validadores específicos por formato
        format_validators = {
            'sire': [
                self._validate_sire_specific,
                self._validate_payment_withholding,
            ],
            'arba': [
                self._validate_arba_specific,
                self._validate_padron_arba,
            ],
            'caba': [
                self._validate_caba_specific,
                self._validate_padron_caba,
            ],
            # ... otras jurisdicciones
        }
        
        return base_validators + format_validators.get(export_format, [])

    def _validate_partner_data(self, options, export_format):
        """Valida datos de partners (CUIT, razón social, etc.)"""
        lines = []
        move_lines = self._get_move_lines(options, export_format)
        
        for line in move_lines:
            partner = line.partner_id
            
            # CUIT obligatorio
            if not partner.vat:
                lines.append(self._create_error_line(
                    'PARTNER_VAT_MISSING',
                    f'CUIT faltante en {partner.name}',
                    f'El partner {partner.name} (ID: {partner.id}) no tiene CUIT configurado. '
                    f'Es obligatorio para exportar retenciones/percepciones.',
                    'error',
                    'missing_data',
                    res_model='res.partner',
                    res_id=partner.id,
                    action_type='fix_required',
                ))
            # Validar formato CUIT
            elif not self._validate_vat_format(partner.vat):
                lines.append(self._create_error_line(
                    'PARTNER_VAT_INVALID',
                    f'CUIT inválido en {partner.name}',
                    f'El CUIT "{partner.vat}" no tiene formato válido (debe ser 11 dígitos).',
                    'error',
                    'invalid_format',
                    res_model='res.partner',
                    res_id=partner.id,
                ))
            
            # Tipo de documento
            if not partner.l10n_latam_identification_type_id:
                lines.append(self._create_error_line(
                    'PARTNER_DOCTYPE_MISSING',
                    f'Tipo de documento faltante en {partner.name}',
                    f'Es necesario especificar el tipo de identificación fiscal.',
                    'warning',
                    'missing_data',
                    res_model='res.partner',
                    res_id=partner.id,
                ))
        
        return lines

    def _validate_tax_configuration(self, options, export_format):
        """Valida configuración de impuestos"""
        lines = []
        move_lines = self._get_move_lines(options, export_format)
        
        for line in move_lines:
            tax = line._get_settlement_tax()
            
            if not tax:
                lines.append(self._create_error_line(
                    'TAX_NOT_FOUND',
                    f'Impuesto no encontrado en línea {line.name}',
                    f'No se pudo determinar el impuesto para la línea de apunte contable.',
                    'error',
                    'tax_config',
                    res_model='account.move.line',
                    res_id=line.id,
                ))
                continue
            
            # Código de régimen para percepciones/retenciones
            if export_format in ['sire', 'arba', 'caba'] and not tax.l10n_ar_code:
                lines.append(self._create_error_line(
                    'TAX_CODE_MISSING',
                    f'Código de régimen faltante en {tax.name}',
                    f'El impuesto {tax.name} debe tener configurado el código de régimen '
                    f'para poder exportarse en formato {export_format.upper()}.',
                    'error',
                    'tax_config',
                    res_model='account.tax',
                    res_id=tax.id,
                    action_type='fix_required',
                ))
        
        return lines

    def _validate_document_integrity(self, options, export_format):
        """Valida integridad de comprobantes"""
        lines = []
        move_lines = self._get_move_lines(options, export_format)
        
        for line in move_lines:
            move = line.move_id
            
            # Estado del comprobante
            if move.state != 'posted':
                lines.append(self._create_error_line(
                    'DOCUMENT_NOT_POSTED',
                    f'Comprobante {move.name} no confirmado',
                    f'El comprobante debe estar confirmado para incluirse en la liquidación.',
                    'warning',
                    'document',
                    res_model='account.move',
                    res_id=move.id,
                ))
            
            # Número de comprobante para percepciones/retenciones aplicadas
            if move.is_invoice() and not move.name or move.name == '/':
                lines.append(self._create_error_line(
                    'DOCUMENT_NUMBER_MISSING',
                    f'Número de comprobante faltante',
                    f'El comprobante debe tener número asignado.',
                    'error',
                    'document',
                    res_model='account.move',
                    res_id=move.id,
                ))
        
        return lines

    def _validate_amounts(self, options, export_format):
        """Valida cálculos y montos"""
        lines = []
        move_lines = self._get_move_lines(options, export_format)
        
        for line in move_lines:
            # Base imponible
            if not line.withholding_id and abs(line.balance) < 0.01:
                lines.append(self._create_error_line(
                    'AMOUNT_ZERO',
                    f'Monto cero en línea {line.name}',
                    f'La línea tiene monto cero. Verificar si debe incluirse.',
                    'info',
                    'computation',
                    res_model='account.move.line',
                    res_id=line.id,
                ))
            
            # Validar base en withholdings
            if line.withholding_id:
                if line.withholding_id.base_amount <= 0:
                    lines.append(self._create_error_line(
                        'WITHHOLDING_BASE_ZERO',
                        f'Base imponible cero en retención {line.withholding_id.name}',
                        f'La retención debe tener base imponible mayor a cero.',
                        'error',
                        'computation',
                        res_model='l10n_ar.payment.withholding',
                        res_id=line.withholding_id.id,
                    ))
        
        return lines

    def explain_retention_application(self, document_id, document_model='account.move'):
        """
        Explica por qué se aplicó o no una retención/percepción.
        
        Returns:
            dict: {
                'applied': bool,
                'reason': str,
                'details': list[dict],
                'related_config': dict,
            }
        """
        document = self.env[document_model].browse(document_id)
        
        explanation = {
            'applied': False,
            'reason': '',
            'details': [],
            'related_config': {},
        }
        
        # Buscar retenciones/percepciones aplicadas
        withholdings = self.env['l10n_ar.payment.withholding'].search([
            ('payment_id.reconciled_invoice_ids', 'in', document.id),
        ])
        
        if withholdings:
            explanation['applied'] = True
            explanation['reason'] = f'Se aplicaron {len(withholdings)} retención(es)'
            
            for wh in withholdings:
                explanation['details'].append({
                    'type': 'retention',
                    'tax': wh.tax_id.name,
                    'base': wh.base_amount,
                    'amount': wh.amount,
                    'aliquot': wh.tax_id.amount,
                    'payment': wh.payment_id.name,
                    'date': wh.payment_id.date,
                })
        else:
            # Analizar por qué no se aplicó
            explanation['applied'] = False
            explanation['reason'] = self._analyze_no_retention_reason(document)
        
        return explanation

    def _analyze_no_retention_reason(self, document):
        """Analiza por qué no se aplicó retención"""
        reasons = []
        
        partner = document.partner_id
        
        # Verificar configuración de partner
        if partner.l10n_ar_gross_income_type == 'exempt':
            reasons.append('El partner está exento de IIBB')
        
        # Verificar régimen del partner
        jurisdiction_ids = partner.l10n_ar_jurisdiction_ids
        if not jurisdiction_ids:
            reasons.append('El partner no tiene jurisdicciones IIBB configuradas')
        
        # Verificar monto mínimo
        # ... lógica adicional ...
        
        return ' | '.join(reasons) if reasons else 'No se encontró configuración que aplique retención'

    def _create_error_line(self, code, title, message, severity, category, 
                          res_model=None, res_id=None, action_type='review'):
        """Helper para crear líneas de validación"""
        return {
            'code': code,
            'title': title,
            'message': message,
            'severity': severity,
            'category': category,
            'res_model': res_model,
            'res_id': res_id,
            'action_type': action_type,
        }
```

#### 2.2.4 Wizard: `l10n_ar.tax.validation.wizard`
**Propósito:** Interfaz interactiva para validación y corrección

```python
class L10nArTaxValidationWizard(models.TransientModel):
    _name = 'l10n_ar.tax.validation.wizard'
    _description = 'Tax Settlement Validation Wizard'

    validation_id = fields.Many2one('l10n_ar.tax.validation.result', required=True)
    report_id = fields.Many2one(related='validation_id.report_id', readonly=True)
    
    # Resumen
    error_count = fields.Integer(related='validation_id.error_count')
    warning_count = fields.Integer(related='validation_id.warning_count')
    info_count = fields.Integer(related='validation_id.info_count')
    validation_state = fields.Selection(related='validation_id.validation_state')
    
    # Líneas filtradas por pestaña
    error_line_ids = fields.One2many(
        'l10n_ar.tax.validation.line', 
        compute='_compute_filtered_lines'
    )
    warning_line_ids = fields.One2many(
        'l10n_ar.tax.validation.line', 
        compute='_compute_filtered_lines'
    )
    info_line_ids = fields.One2many(
        'l10n_ar.tax.validation.line', 
        compute='_compute_filtered_lines'
    )
    
    # Opciones de usuario
    ignore_warnings = fields.Boolean(
        string='Ignorar advertencias y continuar',
        help='Permite exportar incluso con advertencias.'
    )
    
    def action_proceed_export(self):
        """Procede con la exportación si no hay errores críticos"""
        self.ensure_one()
        
        if self.validation_state == 'error' and not self.ignore_warnings:
            raise UserError(
                'No se puede exportar con errores críticos. '
                'Corrija los errores marcados antes de continuar.'
            )
        
        # Retornar a la vista de reporte para continuar exportación
        return {
            'type': 'ir.actions.act_window_close',
        }
    
    def action_fix_issues(self):
        """Intenta corregir automáticamente issues que lo permitan"""
        self.ensure_one()
        
        fixable_lines = self.validation_id.line_ids.filtered(
            lambda l: l.action_type == 'automatic'
        )
        
        for line in fixable_lines:
            try:
                self._apply_automatic_fix(line)
                line.unlink()  # Remover línea corregida
            except Exception as e:
                line.write({
                    'technical_detail': f'Error al corregir automáticamente: {str(e)}',
                    'action_type': 'fix_required',
                })
        
        # Re-validar
        return self.action_revalidate()
    
    def action_revalidate(self):
        """Re-ejecuta validación"""
        # ... lógica de re-validación ...
        return {'type': 'ir.actions.do_nothing'}
```

---

## 3. Integración con Código Existente

### 3.1 Modificaciones en `account.tax.report`

```python
# En account_tax_report.py (heredar en nuevo módulo)

class AccountTaxReport(models.Model):
    _inherit = 'account.tax.report'

    def _l10n_ar_export_file_with_validation(self, options, export_format, export_method):
        """
        Wrapper para exportación con validación previa.
        
        Args:
            export_method: Método original de exportación (ej: 'sire_ret_txt')
        """
        self.ensure_one()
        
        # Ejecutar validación
        validation = self.validate_before_export(options, export_format)
        
        # Si hay errores críticos, mostrar wizard
        if validation.validation_state in ['error', 'warning']:
            return self._show_validation_wizard(validation)
        
        # Si todo OK, proceder con exportación normal
        return getattr(self, export_method)(options)
    
    def _show_validation_wizard(self, validation):
        """Muestra wizard de validación"""
        wizard = self.env['l10n_ar.tax.validation.wizard'].create({
            'validation_id': validation.id,
        })
        
        return {
            'type': 'ir.actions.act_window',
            'name': 'Validación de Liquidación',
            'res_model': 'l10n_ar.tax.validation.wizard',
            'res_id': wizard.id,
            'view_mode': 'form',
            'target': 'new',
            'context': {
                'original_export_method': self._context.get('export_method'),
                'original_options': self._context.get('export_options'),
            },
        }
```

### 3.2 Modificaciones en Reportes Provinciales

**Ejemplo para SIRE:**

```python
# En sire_report.py

class L10nArSireReportHandler(models.AbstractModel):
    _inherit = ['l10n_ar.sire.report.handler', 'l10n_ar.tax.settlement.validation.mixin']
    
    def sire_ret_txt(self, options):
        """Override para agregar validación"""
        export_format = 'sire'
        
        # Ejecutar validación
        validation = self.validate_before_export(options, export_format)
        
        # Si hay errores, mostrar wizard
        if validation.validation_state != 'valid':
            return self._show_validation_wizard(validation)
        
        # Proceder con exportación original
        return self._sire_original_export(options)
    
    def _validate_sire_specific(self, options, export_format):
        """Validaciones específicas de SIRE"""
        lines = []
        move_lines = self._get_move_lines(options, export_format)
        
        for line in move_lines:
            # Validar campos específicos de SIRE
            wh = line.withholding_id
            if not wh:
                continue
            
            # Validar código de impuesto
            if not wh.tax_id.l10n_ar_sire_code:
                lines.append(self._create_error_line(
                    'SIRE_TAX_CODE_MISSING',
                    f'Código SIRE faltante en {wh.tax_id.name}',
                    f'Debe configurar el código SIRE en el impuesto para exportar.',
                    'error',
                    'tax_config',
                ))
        
        return lines
```

---

## 4. Interfaz de Usuario

### 4.1 Vista del Wizard de Validación

**Archivo:** `views/l10n_ar_tax_validation_wizard_view.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Form view del wizard -->
    <record id="view_l10n_ar_tax_validation_wizard_form" model="ir.ui.view">
        <field name="name">l10n_ar.tax.validation.wizard.form</field>
        <field name="model">l10n_ar.tax.validation.wizard</field>
        <field name="arch" type="xml">
            <form string="Validación de Liquidación">
                <header>
                    <button name="action_proceed_export" string="Continuar con exportación" 
                            type="object" class="oe_highlight"
                            invisible="validation_state == 'error'"/>
                    <button name="action_fix_issues" string="Corregir automáticamente" 
                            type="object"/>
                    <button name="action_revalidate" string="Re-validar" type="object"/>
                    <field name="validation_state" widget="statusbar" 
                           statusbar_visible="valid,warning,error"/>
                </header>
                
                <sheet>
                    <!-- Panel de resumen -->
                    <div class="oe_title">
                        <h1>
                            <field name="validation_state" invisible="1"/>
                            <span invisible="validation_state != 'valid'" 
                                  class="badge badge-success">
                                ✓ Validación exitosa
                            </span>
                            <span invisible="validation_state != 'warning'" 
                                  class="badge badge-warning">
                                ⚠ Advertencias detectadas
                            </span>
                            <span invisible="validation_state != 'error'" 
                                  class="badge badge-danger">
                                ✗ Errores críticos
                            </span>
                        </h1>
                    </div>
                    
                    <group>
                        <group>
                            <field name="report_id" readonly="1"/>
                            <field name="validation_id" invisible="1"/>
                        </group>
                        <group>
                            <label for="error_count" string="Resumen"/>
                            <div>
                                <span class="text-danger">
                                    <field name="error_count" class="oe_inline"/> errores
                                </span>
                                 | 
                                <span class="text-warning">
                                    <field name="warning_count" class="oe_inline"/> advertencias
                                </span>
                                 | 
                                <span class="text-info">
                                    <field name="info_count" class="oe_inline"/> informaciones
                                </span>
                            </div>
                        </group>
                    </group>
                    
                    <!-- Notebook con pestañas -->
                    <notebook>
                        <page string="Errores" name="errors" 
                              invisible="error_count == 0">
                            <field name="error_line_ids" nolabel="1">
                                <tree decoration-danger="severity == 'error'" 
                                      create="false" edit="false">
                                    <field name="severity" invisible="1"/>
                                    <field name="category" widget="badge"/>
                                    <field name="title"/>
                                    <field name="message"/>
                                    <field name="action_type" widget="badge"/>
                                    <button name="action_open_record" type="object" 
                                            icon="fa-external-link" 
                                            string="Abrir registro"
                                            invisible="not res_id"/>
                                </tree>
                            </field>
                        </page>
                        
                        <page string="Advertencias" name="warnings" 
                              invisible="warning_count == 0">
                            <field name="warning_line_ids" nolabel="1">
                                <tree decoration-warning="severity == 'warning'" 
                                      create="false" edit="false">
                                    <field name="severity" invisible="1"/>
                                    <field name="category" widget="badge"/>
                                    <field name="title"/>
                                    <field name="message"/>
                                    <field name="action_type" widget="badge"/>
                                    <button name="action_open_record" type="object" 
                                            icon="fa-external-link" 
                                            string="Abrir registro"
                                            invisible="not res_id"/>
                                </tree>
                            </field>
                            <group>
                                <field name="ignore_warnings"/>
                            </group>
                        </page>
                        
                        <page string="Información" name="info" 
                              invisible="info_count == 0">
                            <field name="info_line_ids" nolabel="1">
                                <tree decoration-info="severity == 'info'" 
                                      create="false" edit="false">
                                    <field name="severity" invisible="1"/>
                                    <field name="category" widget="badge"/>
                                    <field name="title"/>
                                    <field name="message"/>
                                </tree>
                            </field>
                        </page>
                        
                        <page string="Detalles técnicos" name="technical" 
                              groups="base.group_no_one">
                            <group>
                                <field name="validation_id"/>
                            </group>
                            <field name="validation_id.line_ids" nolabel="1">
                                <tree>
                                    <field name="code"/>
                                    <field name="severity" widget="badge"/>
                                    <field name="category" widget="badge"/>
                                    <field name="title"/>
                                    <field name="technical_detail"/>
                                </tree>
                            </field>
                        </page>
                    </notebook>
                </sheet>
                
                <footer>
                    <button string="Cerrar" class="btn-secondary" special="cancel"/>
                </footer>
            </form>
        </field>
    </record>
</odoo>
```

### 4.2 Smart Button en Reporte de Liquidación

```xml
<!-- Agregar en vista form del reporte -->
<button name="action_view_last_validation" type="object"
        class="oe_stat_button" icon="fa-check-square">
    <field name="last_validation_count" widget="statinfo" 
           string="Validaciones"/>
</button>
```

### 4.3 Widget de Explicación de Retenciones

**Vista en account.move (factura):**

```xml
<page string="Retenciones y Percepciones" name="tax_withholdings">
    <group>
        <button name="action_explain_retention_application" 
                string="¿Por qué se aplicó/no aplicó retención?" 
                type="object" 
                class="btn-link"
                icon="fa-question-circle"/>
    </group>
    
    <field name="withholding_ids" nolabel="1">
        <tree>
            <field name="tax_id"/>
            <field name="base_amount"/>
            <field name="amount"/>
            <button name="action_explain_this_retention" 
                    type="object" 
                    icon="fa-info-circle" 
                    string="Explicar"/>
        </tree>
    </field>
</page>
```

---

## 5. Configuración y Parámetros del Sistema

### 5.1 Parámetros de Configuración

**Archivo:** `data/ir_config_parameter.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Habilitar/deshabilitar validación automática -->
    <record id="param_enable_auto_validation" model="ir.config_parameter">
        <field name="key">l10n_ar_tax_settlement.enable_auto_validation</field>
        <field name="value">True</field>
    </record>
    
    <!-- Nivel de validación (strict, normal, relaxed) -->
    <record id="param_validation_level" model="ir.config_parameter">
        <field name="key">l10n_ar_tax_settlement.validation_level</field>
        <field name="value">normal</field>
    </record>
    
    <!-- Retener resultados por N días -->
    <record id="param_validation_retention_days" model="ir.config_parameter">
        <field name="key">l10n_ar_tax_settlement.validation_retention_days</field>
        <field name="value">90</field>
    </record>
    
    <!-- Timeout de validación (segundos) -->
    <record id="param_validation_timeout" model="ir.config_parameter">
        <field name="key">l10n_ar_tax_settlement.validation_timeout</field>
        <field name="value">30</field>
    </record>
</odoo>
```

### 5.2 Configuración en res.config.settings

```python
class ResConfigSettings(models.TransientModel):
    _inherit = 'res.config.settings'
    
    l10n_ar_enable_tax_validation = fields.Boolean(
        string='Habilitar validación de liquidaciones',
        config_parameter='l10n_ar_tax_settlement.enable_auto_validation',
    )
    
    l10n_ar_validation_level = fields.Selection([
        ('strict', 'Estricto - Bloquear en cualquier advertencia'),
        ('normal', 'Normal - Advertir pero permitir continuar'),
        ('relaxed', 'Relajado - Solo errores críticos'),
    ], string='Nivel de validación',
        config_parameter='l10n_ar_tax_settlement.validation_level',
        default='normal',
    )
```

---

## 6. Seguridad y Permisos

### 6.1 Grupos de Seguridad

**Archivo:** `security/ir.model.access.csv`

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_tax_validation_result_user,tax.validation.result.user,model_l10n_ar_tax_validation_result,account.group_account_user,1,0,0,0
access_tax_validation_result_manager,tax.validation.result.manager,model_l10n_ar_tax_validation_result,account.group_account_manager,1,1,1,1
access_tax_validation_line_user,tax.validation.line.user,model_l10n_ar_tax_validation_line,account.group_account_user,1,0,0,0
access_tax_validation_line_manager,tax.validation.line.manager,model_l10n_ar_tax_validation_line,account.group_account_manager,1,1,1,1
access_tax_validation_wizard_user,tax.validation.wizard.user,model_l10n_ar_tax_validation_wizard,account.group_account_user,1,1,1,1
```

### 6.2 Record Rules

```xml
<!-- Solo ver validaciones de la compañía del usuario -->
<record id="tax_validation_result_comp_rule" model="ir.rule">
    <field name="name">Tax Validation Result: multi-company</field>
    <field name="model_id" ref="model_l10n_ar_tax_validation_result"/>
    <field name="domain_force">[('company_id', 'in', company_ids)]</field>
</record>
```

---

## 7. Tests Automatizados

### 7.1 Tests Unitarios

**Archivo:** `tests/test_validation_engine.py`

```python
from odoo.tests.common import TransactionCase
from odoo.exceptions import UserError


class TestValidationEngine(TransactionCase):
    
    def setUp(self):
        super().setUp()
        self.company = self.env.company
        self.partner = self.env['res.partner'].create({
            'name': 'Test Partner',
            'vat': '20123456789',
        })
        self.tax = self.env['account.tax'].create({
            'name': 'Retención IIBB',
            'amount': -3.0,
            'type_tax_use': 'purchase',
            'l10n_ar_code': '001',
        })
    
    def test_01_validate_partner_vat_missing(self):
        """Test que detecta CUIT faltante"""
        partner_no_vat = self.env['res.partner'].create({
            'name': 'Partner Sin CUIT',
        })
        
        # Crear factura con partner sin CUIT
        invoice = self.env['account.move'].create({
            'move_type': 'in_invoice',
            'partner_id': partner_no_vat.id,
            # ... otros campos ...
        })
        
        # Ejecutar validación
        report = self.env['account.tax.report'].search([], limit=1)
        validation = report.validate_before_export({}, 'arba')
        
        # Verificar que se detectó el error
        error_lines = validation.line_ids.filtered(
            lambda l: l.code == 'PARTNER_VAT_MISSING'
        )
        self.assertTrue(error_lines, "Debería detectar CUIT faltante")
        self.assertEqual(validation.validation_state, 'error')
    
    def test_02_validate_tax_code_missing(self):
        """Test que detecta código de régimen faltante"""
        tax_no_code = self.env['account.tax'].create({
            'name': 'Impuesto sin código',
            'amount': -2.0,
            'type_tax_use': 'purchase',
            # Sin l10n_ar_code
        })
        
        # ... crear movimiento con este impuesto ...
        
        validation = report.validate_before_export({}, 'sire')
        
        error_lines = validation.line_ids.filtered(
            lambda l: l.code == 'TAX_CODE_MISSING'
        )
        self.assertTrue(error_lines)
    
    def test_03_validation_performance(self):
        """Test de performance - debe validar en <3 segundos"""
        import time
        
        # Crear 100 facturas con retenciones
        invoices = self._create_test_invoices(100)
        
        start = time.time()
        validation = report.validate_before_export({}, 'arba')
        elapsed = time.time() - start
        
        self.assertLess(elapsed, 3.0, 
                       f"Validación tomó {elapsed:.2f}s, debe ser <3s")
    
    def test_04_explain_retention_application(self):
        """Test de explicación de retención aplicada"""
        invoice = self._create_invoice_with_retention()
        
        explanation = self.env['account.tax.report'].explain_retention_application(
            invoice.id, 'account.move'
        )
        
        self.assertTrue(explanation['applied'])
        self.assertIn('details', explanation)
        self.assertTrue(len(explanation['details']) > 0)
```

### 7.2 Tests de Integración

**Archivo:** `tests/test_integration_export.py`

```python
class TestIntegrationExport(TransactionCase):
    
    def test_01_full_validation_export_flow(self):
        """Test del flujo completo: validación -> corrección -> exportación"""
        
        # 1. Crear datos con errores
        partner_no_vat = self.env['res.partner'].create({'name': 'Test'})
        invoice = self._create_invoice(partner_no_vat)
        
        # 2. Intentar exportar - debería mostrar wizard
        report = self.env['account.tax.report'].search([], limit=1)
        result = report.sire_ret_txt({})
        
        # 3. Verificar que se generó validación
        validation = self.env['l10n_ar.tax.validation.result'].search(
            [('report_id', '=', report.id)],
            order='create_date desc',
            limit=1
        )
        self.assertTrue(validation)
        self.assertEqual(validation.validation_state, 'error')
        
        # 4. Corregir datos
        partner_no_vat.vat = '20111222333'
        
        # 5. Re-validar
        validation2 = report.validate_before_export({}, 'sire')
        self.assertEqual(validation2.validation_state, 'valid')
        
        # 6. Exportar exitosamente
        result = report.sire_ret_txt({})
        self.assertIn('file_content', result)
```

---

## 8. Performance y Optimización

### 8.1 Estrategias de Optimización

1. **Caché de validaciones:**
   ```python
   # Cachear resultado si no cambió el periodo/opciones
   cache_key = f"{report.id}_{hash(frozenset(options.items()))}_{export_format}"
   cached = cache.get(cache_key)
   if cached:
       return cached
   ```

2. **Validación en batch:**
   ```python
   # Procesar en lotes de 100 registros
   BATCH_SIZE = 100
   for batch in range(0, len(move_lines), BATCH_SIZE):
       batch_lines = move_lines[batch:batch + BATCH_SIZE]
       # Validar batch
   ```

3. **Lazy loading de detalles técnicos:**
   ```python
   # Solo cargar detalles técnicos cuando se solicitan explícitamente
   technical_detail = fields.Text(compute='_compute_technical_detail')
   ```

### 8.2 Índices de Base de Datos

```python
# En modelo l10n_ar_tax_validation_result
_sql_constraints = [
    ('unique_validation_per_period',
     'UNIQUE(report_id, period_start, period_end, export_format)',
     'Ya existe una validación para este periodo y formato'),
]

# Índices adicionales
def init(self):
    self.env.cr.execute("""
        CREATE INDEX IF NOT EXISTS idx_validation_result_state 
        ON l10n_ar_tax_validation_result (validation_state);
        
        CREATE INDEX IF NOT EXISTS idx_validation_line_severity_category
        ON l10n_ar_tax_validation_line (severity, category);
    """)
```

---

## 9. Plan de Implementación

### 9.1 Fases del Proyecto

#### Fase 1: Fundación (2 semanas)
- [ ] Crear módulo `l10n_ar_tax_settlement_validation`
- [ ] Implementar modelos base (`l10n_ar.tax.validation.result`, `l10n_ar.tax.validation.line`)
- [ ] Implementar mixin de validación
- [ ] Tests unitarios de validadores base

#### Fase 2: Validadores Core (2 semanas)
- [ ] Validador de datos de partners
- [ ] Validador de configuración de impuestos
- [ ] Validador de integridad de documentos
- [ ] Validador de cálculos y montos
- [ ] Tests de validadores

#### Fase 3: Integración con TXT (2 semanas)
- [ ] Integrar con SIRE
- [ ] Integrar con ARBA
- [ ] Integrar con CABA
- [ ] Integrar con Santa Fe
- [ ] Validadores específicos por jurisdicción

#### Fase 4: UI/UX (1 semana)
- [ ] Wizard de validación
- [ ] Vistas de resultados
- [ ] Smart buttons en reportes
- [ ] Widget de explicación de retenciones

#### Fase 5: Testing y Refinamiento (1 semana)
- [ ] Tests de integración completos
- [ ] Tests de performance
- [ ] Ajustes de UX según feedback
- [ ] Optimizaciones

#### Fase 6: Documentación y Rollout (1 semana)
- [ ] Documentación técnica
- [ ] Guía de usuario
- [ ] Video tutorial
- [ ] Release en beta

### 9.2 Criterios de Aceptación

- ✅ Todas las validaciones críticas detectan errores conocidos
- ✅ Tiempo de validación <3s para periodos mensuales típicos
- ✅ Wizard muestra errores de forma clara y accionable
- ✅ Explicación de retenciones cubre casos comunes (>80%)
- ✅ Cobertura de tests >85%
- ✅ Performance no degrada exportación actual
- ✅ Documentación completa y clara

---

## 10. Consideraciones Adicionales

### 10.1 Manejo de Errores

```python
class ValidationTimeoutError(UserError):
    """Error cuando la validación excede el timeout"""
    pass

class ValidationConfigError(UserError):
    """Error de configuración del sistema de validación"""
    pass
```

### 10.2 Logging y Auditoría

```python
import logging
_logger = logging.getLogger(__name__)

def validate_before_export(self, options, export_format):
    _logger.info(
        'Iniciando validación para %s - Formato: %s - Periodo: %s a %s',
        self.report_id.name,
        export_format,
        options['date']['date_from'],
        options['date']['date_to'],
    )
    
    try:
        validation = self._execute_validation(options, export_format)
        _logger.info(
            'Validación completada - Estado: %s - Errores: %d - Advertencias: %d',
            validation.validation_state,
            validation.error_count,
            validation.warning_count,
        )
        return validation
    except Exception as e:
        _logger.error('Error en validación: %s', str(e), exc_info=True)
        raise
```

### 10.3 Extensibilidad para Futuros Casos

```python
# Permitir que otros módulos agreguen validadores
@api.model
def _get_custom_validators(self, export_format):
    """Hook para que otros módulos agreguen validadores"""
    validators = []
    
    # Buscar métodos que sigan el patrón _validate_custom_*
    for attr_name in dir(self):
        if attr_name.startswith('_validate_custom_'):
            validators.append(getattr(self, attr_name))
    
    return validators
```

### 10.4 Notificaciones y Alertas

```python
def _send_validation_alert(self, validation):
    """Enviar notificación si hay errores críticos"""
    if validation.validation_state == 'error':
        # Notificar a usuarios del grupo de contabilidad
        users = self.env.ref('account.group_account_manager').users
        
        for user in users:
            self.env['mail.message'].create({
                'subject': 'Errores críticos en validación de liquidación',
                'body': f'Se detectaron {validation.error_count} errores críticos...',
                'partner_ids': [(4, user.partner_id.id)],
            })
```

---

## 11. Dependencias Externas

### 11.1 Módulos Requeridos

```python
# __manifest__.py
{
    'depends': [
        'account_tax_settlement',           # Base de liquidaciones
        'l10n_ar_account_tax_settlement',   # Liquidación AR
        'l10n_ar_account_reports',          # Reportes provinciales
        'l10n_ar_tax',                      # Impuestos AR
        'account_ux',                       # UX improvements
    ],
}
```

### 11.2 Librerías Python

```python
# requirements.txt (si necesario)
# - No se requieren librerías adicionales para MVP
# - Considerar agregar 'jsonschema' para validación de schemas si se implementa
```

---

## 12. Roadmap Futuro (Post-MVP)

### Versión 2.0
- [ ] IA para sugerir correcciones automáticas
- [ ] Dashboard de tendencias de errores
- [ ] Alertas proactivas (ej: "Padrón desactualizado")
- [ ] Validación en tiempo real al crear retenciones
- [ ] Integración con padrón AFIP para validar CUITs
- [ ] Simulador de liquidación

### Versión 3.0
- [ ] API REST para validación externa
- [ ] App móvil para revisión de validaciones
- [ ] Machine learning para detectar anomalías
- [ ] Integración con sistemas de terceros (estudios contables)

---

## 13. Referencias y Documentación

### 13.1 Documentación Oficial
- [AFIP - SIRE](https://www.afip.gob.ar/sire/)
- [ARBA - Régimen de retención](https://www.arba.gob.ar)
- [Odoo 18 - Account Tax Report](https://github.com/odoo/odoo/tree/18.0/addons/account_tax_settlement)

### 13.2 RFCs y ADRs Relacionados
- RFC-001: Arquitectura de validación de liquidaciones
- ADR-002: Decisión de usar mixin vs herencia múltiple
- ADR-003: Estrategia de caché para validaciones

---

## Aprobaciones

| Rol | Nombre | Firma | Fecha |
|-----|--------|-------|-------|
| Tech Lead | | | |
| Product Owner | | | |
| QA Lead | | | |

---

**Fin del documento**
