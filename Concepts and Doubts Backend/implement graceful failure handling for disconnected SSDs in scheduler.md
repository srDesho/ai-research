# Fix: Scheduler Breaking When SSD Disconnected

## Problema
El scheduler de TBW falla completamente cuando cualquier SSD monitoreado se desconecta, previniendo el registro para todos los demás SSDs.

## Causa Raíz
El método `getTBWFromSMART()` lanza una `RuntimeException` cuando no puede leer de un SSD desconectado. Esta excepción se propaga y marca toda la transacción como rollback-only, rompiendo el scheduler.

## Solución
Implementar un patrón de fallo elegante donde los errores de hardware se manejan como estados válidos del sistema en lugar de excepciones.

### Cambios Clave

#### 1. **HardwareServiceImpl.java** - Retornar valores especiales en lugar de lanzar excepciones
```java
@Override
public long getTBWFromSMART(String ssdModel) {
    try {
        // Intentar leer TBW...
        return tbwInGB;
    } catch (Exception e) {
        logger.warn("Error obteniendo TBW para SSD: {}", ssdModel);
        disableMonitoringForSsd(ssdModel);  // Deshabilitar automáticamente
        return -1;  // Valor especial indicando fallo
    }
}
```

#### 2. **TbwRecordServiceImpl.java** - Manejar valores de fallo elegantemente
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public boolean processSsdRegistration(SSDEntity ssd, LocalDate currentDate, LocalTime currentTime) {
    try {
        long tbw = hardwareService.getTBWFromSMART(ssd.getModel());
        
        if (tbw == -1) {  // Verificar fallo
            logger.warn("Registro TBW omitido para SSD no disponible: {}", ssd.getModel());
            return false;  // Continuar con otros SSDs
        }
        
        // Proceder con registro normal...
        return true;
    } catch (Exception e) {
        logger.error("Error registrando TBW para SSD: {}", ssd.getModel(), e);
        return false;
    }
}
```

## Por Qué Funciona

### 1. **Separa Responsabilidades**
- `HardwareService`: Solo lee datos, retorna valores especiales para fallos
- `TbwRecordService`: Toma decisiones de negocio basadas en validez de datos

### 2. **Transacciones Independientes**
Usar `@Transactional(propagation = Propagation.REQUIRES_NEW)` asegura que cada SSD se procese en su propia transacción. El fallo de uno no afecta a los demás.

### 3. **Recuperación Automática**
Cuando un SSD no puede ser leído:
- El monitoreo se deshabilita automáticamente para ese SSD
- Los demás SSDs continúan procesando normalmente
- El scheduler permanece funcional

## Beneficios
1. **Resiliencia**: Un SSD fallido no rompe todo el sistema
2. **Gestión Automática**: SSDs no disponibles se deshabilitan automáticamente
3. **Continuidad**: Otros SSDs continúan registrando TBW normalmente
4. **Mantenibilidad**: Separación clara entre acceso a hardware y lógica de negocio

## Notas de Implementación
- SSDs fallidos retornan `-1` como valor centinela
- Cada SSD se procesa en transacción independiente
- La deshabilitación automática previene fallos repetidos
- Los logs proveen visibilidad clara del estado del sistema

Esta solución transforma fallos de hardware de excepciones que rompen el sistema a estados de negocio manejables, creando un sistema de monitoreo robusto.

---
**Respuesta para Entrevista**: "Enfrentamos un problema crítico donde un solo SSD desconectado rompía todo el scheduler de monitoreo TBW. En lugar de usar excepciones para fallos de hardware, implementamos un patrón de fallo elegante. El servicio de hardware retorna valores especiales indicando estados de fallo, y cada SSD se procesa en transacciones independientes. Esto creó un sistema resiliente donde los problemas de hardware se manejan como estados de negocio en lugar de excepciones técnicas."

**Commit Message**: `fix: implement graceful failure handling for disconnected SSDs in scheduler`