# HTML Template — EDA Report

Este es el template HTML completo para el reporte EDA. Debe definirse como la variable `HTML_TEMPLATE` en el script Python, usando triple-quote string con `.format()` para los placeholders.

## Cómo usarlo en el script

```python
HTML_TEMPLATE = """<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>EDA Report — {dataset_name}</title>
    <style>
        * {{ box-sizing: border-box; margin: 0; padding: 0; }}
        body {{ font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
               background: #f1f5f9; color: #1e293b; line-height: 1.6; }}
        .container {{ max-width: 1100px; margin: 0 auto; padding: 2rem 1.5rem; }}

        /* Header */
        header {{ background: linear-gradient(135deg, #1e293b 0%, #334155 100%);
                  color: white; padding: 2rem 2.5rem; border-radius: 12px;
                  margin-bottom: 2rem; }}
        header h1 {{ font-size: 1.7rem; font-weight: 700; margin-bottom: 0.4rem; }}
        header .meta {{ color: #94a3b8; font-size: 0.9rem; display: flex; gap: 1.5rem; flex-wrap: wrap; }}
        header .meta span {{ display: flex; align-items: center; gap: 0.3rem; }}

        /* Table of Contents */
        .toc {{ background: white; padding: 1.5rem 2rem; border-radius: 12px;
                margin-bottom: 2rem; box-shadow: 0 1px 4px rgba(0,0,0,0.07); }}
        .toc h2 {{ font-size: 0.8rem; text-transform: uppercase; letter-spacing: 0.08em;
                   color: #64748b; margin-bottom: 1rem; }}
        .toc ol {{ padding-left: 1.3rem; columns: 2; column-gap: 2rem; }}
        .toc li {{ margin: 0.35rem 0; font-size: 0.95rem; }}
        .toc a {{ color: #3b82f6; text-decoration: none; }}
        .toc a:hover {{ text-decoration: underline; }}

        /* Sections */
        .section {{ background: white; border-radius: 12px; padding: 2rem;
                    margin-bottom: 1.5rem; box-shadow: 0 1px 4px rgba(0,0,0,0.07); }}
        .section h2 {{ font-size: 1.2rem; color: #0f172a; margin-bottom: 1rem;
                       padding-bottom: 0.6rem; border-bottom: 2px solid #e2e8f0;
                       display: flex; align-items: center; gap: 0.5rem; }}
        .section h3 {{ font-size: 1rem; color: #475569; margin: 1.2rem 0 0.6rem; }}

        /* Narrative box */
        .narrative {{ background: #f0f9ff; border-left: 4px solid #3b82f6;
                      padding: 0.9rem 1.2rem; border-radius: 0 8px 8px 0;
                      margin-bottom: 1.5rem; font-size: 0.95rem; color: #1e40af; }}

        /* Alert narrative (warnings) */
        .narrative.warning {{ background: #fffbeb; border-left-color: #f59e0b; color: #92400e; }}
        .narrative.danger  {{ background: #fff1f2; border-left-color: #ef4444; color: #991b1b; }}
        .narrative.success {{ background: #f0fdf4; border-left-color: #22c55e; color: #166534; }}

        /* Charts */
        .chart-wrap {{ margin: 1rem 0; text-align: center; }}
        .chart-wrap img {{ max-width: 100%; border-radius: 8px;
                           border: 1px solid #e2e8f0; }}
        .chart-grid {{ display: grid; grid-template-columns: repeat(auto-fit, minmax(380px, 1fr));
                       gap: 1rem; margin: 1rem 0; }}
        .chart-grid .chart-wrap img {{ width: 100%; }}

        /* Tables */
        .table-wrap {{ overflow-x: auto; margin: 1rem 0; }}
        .stats-table {{ width: 100%; border-collapse: collapse; font-size: 0.88rem; }}
        .stats-table th {{ background: #f8fafc; text-align: left; padding: 0.55rem 0.9rem;
                           border-bottom: 2px solid #e2e8f0; color: #475569;
                           font-weight: 600; white-space: nowrap; }}
        .stats-table td {{ padding: 0.45rem 0.9rem; border-bottom: 1px solid #f1f5f9;
                           white-space: nowrap; }}
        .stats-table tr:hover td {{ background: #f8fafc; }}
        .stats-table tr:last-child td {{ border-bottom: none; }}

        /* Badges */
        .badge {{ display: inline-block; padding: 0.15rem 0.55rem; border-radius: 999px;
                  font-size: 0.78rem; font-weight: 600; }}
        .badge-ok      {{ background: #dcfce7; color: #166534; }}
        .badge-warning {{ background: #fef9c3; color: #854d0e; }}
        .badge-danger  {{ background: #fee2e2; color: #991b1b; }}

        /* Suggestions box */
        .suggestions {{ background: #fafafa; border: 1px solid #e2e8f0; border-radius: 10px;
                        padding: 1.5rem; margin-top: 1rem; }}
        .suggestions h3 {{ color: #475569; font-size: 0.95rem; margin-bottom: 0.8rem; }}
        .suggestions ul {{ padding-left: 1.4rem; }}
        .suggestions li {{ margin: 0.5rem 0; font-size: 0.93rem; }}
        .suggestions li strong {{ color: #1e293b; }}

        /* Conclusions */
        .conclusion-grid {{ display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
                            gap: 1rem; margin: 1rem 0; }}
        .conclusion-card {{ background: #f8fafc; border-radius: 10px; padding: 1.2rem;
                            border: 1px solid #e2e8f0; }}
        .conclusion-card h4 {{ font-size: 0.8rem; text-transform: uppercase;
                               letter-spacing: 0.06em; color: #64748b; margin-bottom: 0.6rem; }}
        .conclusion-card p  {{ font-size: 0.93rem; color: #334155; }}

        /* Footer */
        footer {{ text-align: center; color: #94a3b8; font-size: 0.82rem;
                  padding: 2rem 0 1rem; }}
    </style>
</head>
<body>
<div class="container">

    <header>
        <h1>📊 Reporte EDA — {dataset_name}</h1>
        <div class="meta">
            <span>📅 {date}</span>
            <span>📋 {n_rows:,} filas</span>
            <span>🔢 {n_cols} columnas</span>
            <span>🎯 {target_info}</span>
        </div>
    </header>

    <div class="toc">
        <h2>Tabla de Contenidos</h2>
        <ol>
            <li><a href="#overview">Overview del Dataset</a></li>
            <li><a href="#quality">Calidad de Datos</a></li>
            <li><a href="#descriptive">Estadísticas Descriptivas</a></li>
            <li><a href="#univariate">Análisis Univariado</a></li>
            <li><a href="#outliers">Detección de Outliers</a></li>
            <li><a href="#bivariate">Análisis Bivariado</a></li>
            <li><a href="#multivariate">Análisis Multivariado</a></li>
            <li><a href="#features">Feature Engineering</a></li>
            {time_series_toc}
            <li><a href="#suggestions">Análisis Sugeridos</a></li>
            <li><a href="#conclusions">Conclusiones y Próximos Pasos</a></li>
        </ol>
    </div>

    {content}

    <footer>
        Reporte generado automáticamente · EDA Skill · Claude
    </footer>

</div>
</body>
</html>"""
```

---

## Función auxiliar para construir secciones

Incluir esta función en el script para estandarizar la generación de secciones:

```python
def build_section(section_id, title, emoji, narrative, charts=None,
                  tables=None, extra_html='', narrative_type=''):
    """
    Construye una sección HTML del reporte.

    Args:
        section_id   : str  — ID del anchor (ej. 'quality')
        title        : str  — Título de la sección
        emoji        : str  — Emoji del título (ej. '🔍')
        narrative    : str  — Texto interpretativo de la sección
        charts       : list — Lista de strings base64 (data:image/png;base64,...)
        tables       : list — Lista de DataFrames a convertir a HTML
        extra_html   : str  — HTML adicional (ej. sección de sugerencias)
        narrative_type: str — '' | 'warning' | 'danger' | 'success'
    """
    nav_class = f'narrative {narrative_type}'.strip()
    html = f'<div class="section" id="{section_id}">\n'
    html += f'  <h2>{emoji} {title}</h2>\n'
    html += f'  <div class="{nav_class}">{narrative}</div>\n'

    if charts:
        if len(charts) == 1:
            html += f'  <div class="chart-wrap"><img src="{charts[0]}"/></div>\n'
        else:
            html += '  <div class="chart-grid">\n'
            for c in charts:
                html += f'    <div class="chart-wrap"><img src="{c}"/></div>\n'
            html += '  </div>\n'

    if tables:
        for df_table in tables:
            html += f'  <div class="table-wrap">{df_to_html(df_table)}</div>\n'

    html += extra_html
    html += '</div>\n'
    return html
```

---

## Ejemplo de uso en el script

```python
# Fase 2 — Calidad
narrative_quality = (
    f"El dataset contiene <strong>{null_pct[null_pct > 0].shape[0]} columnas con valores faltantes</strong>. "
    f"La columna con más nulos es <code>{null_pct.idxmax()}</code> con un "
    f"{null_pct.max():.1f}% de valores ausentes. "
    f"Se encontraron <strong>{duplicates} filas duplicadas</strong> "
    f"({duplicates/n_rows*100:.1f}% del total)."
)

sections.append(build_section(
    section_id='quality',
    title='Calidad de Datos',
    emoji='🔍',
    narrative=narrative_quality,
    tables=[null_summary_df],  # incluye columnas: nulos (#), nulos (%) — sin badge de severidad
    narrative_type='warning' if null_pct.max() > 20 else ''
))
```

---

## Sección de Conclusiones (estructura sugerida)

```python
conclusions_html = """
<div class="conclusion-grid">
    <div class="conclusion-card">
        <h4>📋 Resumen del Dataset</h4>
        <p>{summary_text}</p>
    </div>
    <div class="conclusion-card">
        <h4>⚠️ Alertas de Calidad</h4>
        <p>{quality_alerts}</p>
    </div>
    <div class="conclusion-card">
        <h4>🏆 Variables Más Relevantes</h4>
        <p>{top_vars}</p>
    </div>
    <div class="conclusion-card">
        <h4>🚀 Próximos Pasos</h4>
        <p>{next_steps}</p>
    </div>
</div>
"""
```
