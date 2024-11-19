# Seguridad en Contenedores: Capabilities y Escenarios de Escape
## Una Guía Práctica de Seguridad

### Prerequisitos
- Dockerhub account
- Laptop with web brower
- Docker basics

### Sección 1: Introducción (5 minutos)
- ¿Por qué esta presentación?
- Objetivos de aprendizaje
- Advertencias de seguridad
- Configuración del laboratorio

### Sección 2: Fundamentos de Seguridad (10 minutos)

#### 2.1 Componentes Básicos
- Namespaces
- cgroups
- Capabilities
- SecurityContexts

#### 2.2 ¿Qué son los Linux Capabilities?
```bash
# Ejemplo: Ver capabilities actuales
capsh --print

# Ejemplo: Contenedor sin capabilities
docker run --cap-drop=ALL nginx
```

### Sección 3: Linux Capabilities en Detalle (15 minutos)

#### 3.1 Capabilities Críticos
```bash
# Capabilities más peligrosos:
CAP_SYS_ADMIN        # Operaciones administrativas del sistema
CAP_NET_ADMIN        # Configuración de red
CAP_SYS_MODULE       # Cargar módulos del kernel
CAP_SYS_PTRACE       # Depurar procesos
CAP_SYS_CHROOT       # Usar chroot()
CAP_NET_RAW          # Usar raw sockets
CAP_SETUID           # Cambiar UID
CAP_SETGID           # Cambiar GID
CAP_MKNOD            # Crear archivos especiales
CAP_AUDIT_WRITE      # Escribir registros de auditoría
CAP_AUDIT_CONTROL    # Configurar auditoría
```

#### 3.2 Laboratorio: Explorando Capabilities
```bash
# Ver capabilities de un contenedor
docker inspect container_name | grep -A 10 CapAdd

# Agregar capabilities específicos
docker run --cap-add=SYS_ADMIN ubuntu

# Quitar todos y agregar solo los necesarios
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

#### Docker Lab:

- [security-capabilities](https://training.play-with-docker.com/security-capabilities/)

### Sección 4: Escenarios Vulnerables Comunes (20 minutos)

#### 4.1 Docker in Docker (DinD)
```yaml
# ❌ Configuración vulnerable común
version: '3'
services:
  dind-vulnerable:
    image: docker:dind
    privileged: true
    volumes:
      - ./:/workspace
```

##### Demostración de Escape
```bash
# Desde dentro del contenedor DinD
docker run -v /:/host -it ubuntu  # Acceso total al host

# O peor aún, crear un contenedor privilegiado
docker run --privileged -it ubuntu

# Acceder al kernel del host
mkdir /tmp/cgroup
mount -t cgroup cgroup /tmp/cgroup
mkdir /tmp/cgroup/x
echo 1 > /tmp/cgroup/x/notify_on_release
```

#### 4.2 Jenkins con Docker
```yaml
# ❌ Jenkins con acceso inseguro a Docker
services:
  jenkins:
    image: jenkins/jenkins
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - jenkins_home:/var/jenkins_home
```

##### Demostración de Escape
```bash
# Desde dentro de Jenkins
docker run -v /:/hostfs alpine chroot /hostfs /bin/sh
```

#### 4.3 Desarrollo Local Inseguro
```bash
# ❌ Montajes peligrosos comunes
docker run -v ~/.ssh:/root/.ssh
docker run -v ~/.aws:/root/.aws
```

```bash
# Depuración con capabilities excesivos
docker run --cap-add=SYS_PTRACE -it ubuntu
docker run --cap-add=SYS_ADMIN -it ubuntu
```

### Sección 5: Técnicas de Escape (15 minutos)

#### 5.1 A través de Capabilities
```bash
# Ejemplo con CAP_SYS_ADMIN
docker run --cap-add=SYS_ADMIN -it ubuntu bash
mkdir /tmp/cgroup
mount -t cgroup cgroup /tmp/cgroup
# Continuar con ejemplo de escape...
```

#### 5.2 A través de Montajes
```bash
# Acceso al host mediante montajes
docker run -v /:/host -it ubuntu
cd /host && chroot .
```

#### 5.3 A través del Socket de Docker
```bash
# Usando el socket expuesto
docker run -v /var/run/docker.sock:/var/run/docker.sock -it ubuntu
# Instalar Docker CLI
apt update && apt install -y docker.io
# Crear nuevo contenedor privilegiado
docker run --privileged -it ubuntu
```

### Sección 6: Mitigaciones y Mejores Prácticas (15 minutos)

#### 6.1 Configuración Segura de DinD
```yaml
# ✅ DinD más seguro
version: '3'
services:
  dind-secure:
    image: docker:dind
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - SETGID
      - SETUID
    read_only: true
```

#### 6.2 Alternativas Seguras
```yaml
# ✅ Docker out of Docker (DooD)
services:
  dood:
    image: docker:latest
    user: "${UID}:${DOCKER_GID}" # Usuario no root con acceso a docker
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

#### 6.3 Jenkins Seguro
```yaml
# ✅ Jenkins con configuración segura
services:
  jenkins-secure:
    image: jenkins/jenkins
    user: jenkins
    group_add:
      - "${DOCKER_GID}"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

### Seccion 7: Herramientas de auditoria
```bash
# Analizar capabilities de un contenedor
docker inspect <container_id> | jq '.[0].HostConfig.CapAdd'

# Verificar montajes
docker inspect <container_id> | jq '.[0].Mounts'

# Verificar privilegios
docker inspect <container_id> | jq '.[0].HostConfig.Privileged'
```

### Sección 8: Lista de Verificación de Seguridad (5 minutos)

#### Checklist Diario
- [ ] No usar --privileged
- [ ] Limitar capabilities
- [ ] Evitar montar docker.sock
- [ ] Usar usuarios no root
- [ ] Implementar read-only cuando sea posible
- [ ] Escanear imágenes regularmente
- [ ] Monitorear actividad sospechosa

