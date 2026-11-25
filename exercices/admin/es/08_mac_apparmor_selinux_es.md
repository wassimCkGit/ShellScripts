# 🧱 A8 · AppArmor / SELinux

*Capítulo [05 · Seguridad](../../05_securite.md) — parte **AppArmor / SELinux***

> [!NOTE]
> **Objetivos**: comprender qué añade un control de acceso obligatorio (MAC) encima de los permisos POSIX,
> saber en qué modo está en ejecución la máquina, leer un rechazo en los registros, ajustar un perfil o un
> contexto — y sobre todo **no desactivar la protección en la primera dificultad**.

> [!IMPORTANT]
> Debian / Ubuntu usan **AppArmor**, RHEL / Fedora / Rocky usan **SELinux**. Haga la parte que
> corresponde a su VM, luego **lea** la otra: se encontrará con ambas en su carrera.

---

## Ejercicio 8.1 — El principio

1. En una oración cada una: ¿qué es el **DAC** (los permisos POSIX del capítulo 02) y qué es el **MAC**?
2. Un proceso se ejecuta como `root`. DAC le permite leer todo. ¿Qué más puede evitarlo?
3. Dé un ejemplo concreto de compromiso que un MAC bloquea y que un `chmod` correcto no hubiera bloqueado.
4. En su máquina: ¿está AppArmor presente? ¿activo? ¿Y SELinux? ¿Qué órdenes le lo dicen?

---

## Parte AppArmor *(Debian, Ubuntu, SUSE)*

### Ejercicio 8.2 — Estado y modos

1. Muestre el estado de AppArmor: ¿cuántos perfiles cargados, cuántos en `enforce`, cuántos en `complain`?
2. ¿Cuál es la diferencia entre `enforce`, `complain` y `disabled`? ¿En qué orden se usan
   cuando se construye un perfil?
3. Liste los perfiles cargados y ubique el de su servidor web o su cliente de base de datos.
4. Pase un perfil existente a `complain`, verifique el cambio, luego vuelva a `enforce`.

### Ejercicio 8.3 — Leer un rechazo

1. Provoque un rechazo: haga leer por un programa confinado un archivo al que su perfil no le da acceso
   (por ejemplo, haga servir por nginx una carpeta fuera de su perfil).
2. Encuentre el rechazo en los registros. ¿Qué palabra clave busca?
   *(Capítulo [03](../../03_commandes_essentielles.md) para el filtrado.)*
3. Decodifique la línea: ¿qué programa, qué operación, qué ruta, qué perfil?
4. Un usuario le dice "funciona como root pero no desde el servicio" aunque los permisos POSIX son
   correctos. ¿Cuál es su primera hipótesis, y cómo la verifica en treinta segundos?

### Ejercicio 8.4 — Escribir y ajustar un perfil

1. Escriba `/usr/local/bin/backup.sh`, un script que lee `/srv/ticketflow` y escribe en
   `/srv/ticketflow/backups`.
2. Cree un perfil para este script: solo lectura en las fuentes, escritura en el destino,
   ejecución de los binarios que necesita, y **nada más**.
3. Cárguelo en modo `complain`, use el script en **todos** sus casos de uso (incluyendo errores),
   luego recolecte lo que faltó.
4. Pase a `enforce` y verifique que el script sigue funcionando.
5. Verifique que **no puede** leer `/etc/shadow` ni escribir fuera de su carpeta destino.
6. Modifique el perfil de un paquete (nginx, por ejemplo) para autorizar un docroot personalizado —
   **sin** editar el archivo proporcionado por el paquete. ¿Dónde escribe esta sobrecarga?
7. Verifique la sintaxis de un perfil sin cargarlo, luego recárguelo, luego recargue el conjunto.

---

## Parte SELinux *(RHEL, Fedora, Rocky)*

### Ejercicio 8.5 — Estado y modos

1. Muestre el modo actual y el modo configurado en el arranque. ¿Dónde se configura este último?
2. Pase temporalmente a `permissive`, verifique, vuelva a `enforcing`.
   ¿Por qué **nunca** hace un `SELINUX=disabled` definitivo para "arreglar" un problema?
3. ¿Cuál es la diferencia entre `permissive` y `disabled` desde el punto de vista del diagnóstico?

### Ejercicio 8.6 — Contextos

1. Muestre el contexto de seguridad de `/var/www/html`, de un proceso `httpd` y de su propio shell.
2. ¿Qué significan los cuatro campos de un contexto?
3. Mueva un archivo desde su *home* a `/var/www/html` con `mv`, luego con `cp`.
   Compare los contextos obtenidos y explique la trampa clásica.
4. Repare el contexto del archivo movido — primero puntualmente, luego **duradivamente** declarando
   la regla para la ruta y reaplicándola.
5. Haga servir por su servidor web un docroot en `/srv/web`: ¿qué tipo hay que declarar, y cómo?

### Ejercicio 8.7 — Booleanos, puertos y diagnóstico

1. Liste los booleanos activos concernientes a su servidor web. ¿Qué hace `httpd_can_network_connect`?
2. Active uno de forma **permanente**, y explique qué cambia la opción de persistencia.
3. Haga escuchar `sshd` en el puerto 2222: ¿qué tipo de puerto hay que declarar, y con qué orden?
   *(El vínculo con el ejercicio [7.3](07_ssh_et_pare_feu_es.md) es directo.)*
4. Provoque un rechazo, encuentre en los registros de auditoría, y haga que le explique la causa.
5. Genere el módulo de política que autorizaría este rechazo — **y léalo antes de instalarlo**.
   ¿Por qué nunca se instala sin leerlo?

---

## Ejercicio 8.8 — El método de diagnóstico

Un servicio rechaza acceso a un archivo. Escriba, en `journal.md`, su **árbol de decisión** partiendo
de la pregunta "¿es un problema de permisos?":

1. ¿Qué verifica primero (DAC: propietario, permisos, ACL, ruta atravesable)?
2. ¿Cómo sabe, en una orden, si MAC es responsable?
3. ¿Cómo lo confirma **sin** desactivar la protección?
4. ¿Cómo lo arregla correctamente — y por qué la corrección "pongo todo en 777" o
   "desactivo SELinux" es un aviso de fracaso?
5. ¿Qué escribe en el registro una vez resuelto el problema?

---

## ✅ Verificación

- Sabe decir, en cualquier máquina, si un MAC está en ejecución y en qué modo.
- Su script de respaldo se ejecuta bajo un perfil / contexto restringido, **y** ha probado que no
  puede leer `/etc/shadow`.
- Ha leído y decodificado al menos un rechazo en los registros, y corregido la causa en lugar de desactivar la protección.
- Su árbol de decisión del ejercicio 8.8 cabe en una página y se lee por alguien más.
