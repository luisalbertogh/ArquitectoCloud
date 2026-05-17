---
post_title: Prácticas básicas de AWS CloudWatch a través de la consola
author1: Luis Alberto Garcia Hernandez
post_slug: aws-cloudwatch-practicas-basicas
microsoft_alias: luis.garcia
featured_image: ""
categories:
  - Cloud
  - AWS
  - Monitoring
tags:
  - AWS
  - CloudWatch
  - Monitoring
  - Observability
ai_note: true
summary: >
  Guía práctica con 5 ejercicios básicos para demostrar las características
  fundamentales de AWS CloudWatch a través de la consola de AWS. Cada práctica
  se puede completar en 5-6 minutos.
post_date: 2026-05-17
---

# Prácticas básicas de AWS CloudWatch

Esta guía incluye 5 prácticas fundamentales para comprender y utilizar AWS CloudWatch a través de la consola de AWS. Cada práctica está diseñada para completarse en 5-6 minutos, permitiendo acabar todas en menos de media hora.

## 1. Explorar el Dashboard predeterminado y crear métricas personalizadas

**Objetivo:** Familiarizarse con la interfaz de CloudWatch y entender qué son las métricas.

**Pasos:**

1. Inicia sesión en la [Consola de AWS](https://console.aws.amazon.com/)
2. Navega a **CloudWatch** en el menú de servicios
3. En el panel izquierdo, selecciona **Dashboards** > **Dashboard predeterminado**
4. Observa las métricas disponibles de los recursos existentes (EC2, RDS, etc.)
5. Si no hay dashboards, ve a **Dashboards** > **Crear dashboard**
6. Nombra el dashboard como `Dashboard-Practicas`
7. Selecciona el widget tipo "Métrica" y elige una métrica de un servicio disponible (por ejemplo, CPU de EC2)

**Resultado esperado:** Entender cómo se visualizan las métricas en CloudWatch.

---

## 2. Crear una alarma simple basada en métricas

**Objetivo:** Aprender a crear alarmas para monitorear cambios en métricas clave.

**Pasos:**

1. En CloudWatch, selecciona **Alarmas** > **Todas las alarmas**
2. Haz clic en **Crear alarma**
3. Selecciona **Seleccionar métrica**
4. Elige un servicio (por ejemplo, **EC2** > **Por instancia** > selecciona una instancia)
5. Selecciona la métrica "CPU Utilization"
6. Configura los umbrales:
   - Umbral: Superior a 70%
   - Período de evaluación: 1 minuto
   - Número de períodos: 1
7. Configura la acción para la alarma:
   - Selecciona **En estado de alarma** > **Enviar notificación a SNS**
   - Si no tienes un tema SNS, crea uno nuevo llamado `CloudWatch-Alerts`
8. Nombra la alarma como `Alarma-CPU-Alta`
9. Haz clic en **Crear alarma**

**Resultado esperado:** Una alarma activa que monitoreará la métrica de CPU.

---

## 3. Ver y analizar logs desde CloudWatch Logs

**Objetivo:** Comprender cómo centralizarse los logs de diferentes servicios en CloudWatch.

**Pasos:**

1. Navega a **CloudWatch** > **Logs** > **Grupos de logs**
2. Observa los grupos de logs disponibles (pueden ser de Lambda, aplicaciones, etc.)
3. Si no hay logs, crea un grupo de logs manual:
   - Haz clic en **Crear grupo de logs**
   - Nombra el grupo como `Practica-Logs`
   - Configura la retención a 1 día
   - Haz clic en **Crear**
4. Dentro del grupo, haz clic en **Crear stream de logs** y nombralo `stream-1`
5. Selecciona el stream y haz clic en **Insertar eventos de registro**
6. Añade algunos eventos de ejemplo (ej: "Aplicación iniciada" o "Error de conexión")
7. Visualiza los eventos en el stream y su marca de tiempo

**Resultado esperado:** Entender cómo se organizan y visualizan los logs en CloudWatch.

---

## 4. Crear un filtro de métrica para extraer datos de logs

**Objetivo:** Aprender a convertir datos de logs en métricas personalizadas.

**Pasos:**

1. Ve a **CloudWatch** > **Logs** > **Grupos de logs**
2. Selecciona un grupo de logs existente (o usa el creado en la práctica anterior)
3. Haz clic en **Acciones** > **Crear filtro de métricas**
4. En el campo "Patrón de filtro", introduce: `[ERROR]` (para capturar líneas con ERROR)
5. Haz clic en **Siguiente**
6. Configura los detalles de la métrica:
   - Nombre del espacio de nombres: `Aplicacion-Practicas`
   - Nombre de la métrica: `Contador-Errores`
   - Valor de métrica: `1`
7. Haz clic en **Crear filtro de métricas**
8. Verifica que el filtro aparece en la lista del grupo de logs

**Resultado esperado:** Una métrica personalizada que cuenta eventos de error en los logs.

---

## 5. Configurar un widget de visualización y personalizar un dashboard

**Objetivo:** Practicar la creación de dashboards personalizados para monitoreo visual.

**Pasos:**

1. Ve a **CloudWatch** > **Dashboards**
2. Abre o crea un dashboard llamado `Dashboard-Personalizado`
3. Haz clic en **Editar**
4. Añade varios widgets:
   - **Widget 1 (Métrica):** Selecciona una métrica de EC2 (ej: CPU Utilization)
   - **Widget 2 (Métrica numérica):** Visualiza la métrica en formato de número
   - **Widget 3 (Logs):** Añade un widget de logs de un grupo existente
5. Personaliza el layout:
   - Redimensiona los widgets arrastrándolos
   - Cambia el período de tiempo a "Última 1 hora"
   - Añade un título al dashboard
6. Haz clic en **Guardar dashboard**

**Resultado esperado:** Un dashboard personalizado con múltiples visualizaciones.

---

## Conceptos clave aprendidos

- **Métricas:** Datos numéricos que CloudWatch recopila y almacena
- **Alarmas:** Notificaciones automáticas basadas en umbrales de métricas
- **Logs:** Registros de aplicaciones centralizados en CloudWatch
- **Filtros de métricas:** Conversión de datos de logs en métricas personalizadas
- **Dashboards:** Visualizaciones personalizadas para monitoreo

---

## Limpieza de recursos (opcional)

Si deseas limpiar los recursos creados:

1. Elimina las alarmas en **CloudWatch** > **Alarmas** > **Todas las alarmas**
2. Elimina los grupos de logs en **CloudWatch** > **Logs** > **Grupos de logs**
3. Elimina los dashboards en **CloudWatch** > **Dashboards**
4. Elimina el tema SNS en **SNS** > **Temas** si lo creaste
