# Objetivo 1: Estandarización de Vulnerabilidades (CVE y CWE)


## Conceptos y Diferencia Fundamental
*   **CVE (Common Vulnerabilities and Exposures):** Es un identificador único asignado a una vulnerabilidad de seguridad específica y pública encontrada en un producto real (ej. `CVE-2021-44228` en Log4j).
*   **CWE (Common Weakness Enumeration):** Es un catálogo que clasifica los *tipos* de debilidades o errores de diseño/código que causan vulnerabilidades (ej. `CWE-89` para Inyección SQL).
*   **Diferencia:** El **CWE** es la categoría del error (la "enfermedad"), mientras que el **CVE** es la instancia específica de ese error en un software (el "caso clínico").

## Asignación y Consulta
*   **¿Quién asigna los CVE?** Los **CNAs** (CVE Numbering Authorities). Son organizaciones (empresas de software, investigadores, etc.) autorizadas por MITRE para asignar identificadores CVE dentro de su ámbito.
*   **¿Dónde se consultan?** Las bases de datos públicas oficiales son la **NVD (National Vulnerability Database)** del NIST y el portal oficial de **MITRE**.

## Herramientas de la Industria
Para detectar vulnerabilidades con CVEs asignados de forma automatizada, la industria utiliza escáneres de vulnerabilidad como:
1.  **Tenable Nessus:** Escáner líder para redes y sistemas operativos.
2.  **Qualys Vulnerability Management:** Plataforma en la nube para escaneo continuo de infraestructura.