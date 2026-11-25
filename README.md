# Migración gradual de PHP 5.6 a PHP 8.4 con k3s

Prueba de concepto que demuestra cómo ejecutar un sistema PHP legacy (5.6)
junto con su versión migrada (8.4) en un clúster Kubernetes (k3s) sobre
Debian 13, moviendo el tráfico por ruta, módulo por módulo.

> **Aviso:** este repositorio usa una aplicación de prueba.
> No contiene código, datos ni credenciales de ningún sistema real.

## Objetivo

Evitar una migración "big bang". Se despliegan ambas versiones en paralelo y
se redirige cada ruta a PHP 8 conforme se migra.

## Arquitectura

```
Cliente → Traefik (Ingress)
            ├── /nuevo → Service app-nuevo  → Pod PHP 8.4
            └── /      → Service app-legacy → Pod PHP 5.6
```

| Componente | Función |
|---|---|
| `Deployment` | Mantiene los pods de cada versión |
| `Service` | Nombre estable interno para cada versión |
| `Ingress` (Traefik) | Enruta por ruta: `/nuevo` a PHP 8, el resto a PHP 5 |

## Estructura

```
.
├── legacy/          # App de prueba PHP 5.6 + Dockerfile
├── app/             # App de prueba PHP 8.4 + Dockerfile
├── miapp.yaml       # Manifiestos de Kubernetes
└── README.md
```

## Requisitos

- Debian 13
- k3s (incluye containerd y Traefik)
- Docker (solo para construir imágenes)

## Cómo reproducirlo

```bash
# 1. Instalar k3s
curl -sfL https://get.k3s.io | sh -
kubectl get nodes

# 2. Construir las imágenes
docker build -t miapp-legacy:1.0 ./legacy
docker build -t miapp-nuevo:1.0 ./app

# 3. Importarlas a k3s
docker save miapp-legacy:1.0 | k3s ctr images import -
docker save miapp-nuevo:1.0  | k3s ctr images import -

# 4. Desplegar
kubectl apply -f miapp.yaml
kubectl get pods,svc,ingress

# 5. Probar
curl -H "Host: miapp.example.com" http://localhost/
curl -H "Host: miapp.example.com" http://localhost/nuevo/
```

Resultado esperado:

```
<h1>LEGACY - PHP 5.6.x</h1>
<h1>NUEVO - PHP 8.4.x</h1>
```

## Problemas encontrados

- Archivo movido a una ruta incorrecta (`app/index.php` en lugar de `app/nuevo/index.php`).
- Error de sangría en el `Ingress` (`port` debe ser un objeto con `number`).
- Campo `ports` anidado de más en un `Service`.

## Limitaciones

- PHP 5.6 no recibe parches de seguridad desde 2018: aislar y no exponer a internet.
- La app de prueba no incluye base de datos, sesiones compartidas ni archivos subidos.

## Próximos pasos

- [ ] Base de datos y Redis (sesiones compartidas)
- [ ] Volumen para archivos subidos
- [ ] Secrets y ConfigMaps
- [ ] Pruebas de resiliencia y rollback
- [ ] Análisis de compatibilidad con PHPCompatibility y Rector
