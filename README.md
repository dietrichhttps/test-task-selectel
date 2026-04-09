# Тестовое задание — Системное администрирование

## 1) Подсчёт суммарного объёма данных по IP

Используйте один из однострочников для анализа access.log:

**robust** (работает с common/combined форматом):
```bash
awk '{ match($0, /" [0-9]{3} ([0-9-]+)/, a); b=a[1]; if(b=="" || b=="-") b=0; sum[$1]+=b } END{ for (i in sum) print sum[i] " bytes for " i }' access.log | sort -nr
```

**простой** (если байты — 10-е поле):
```bash
awk '{ if($10=="-") b=0; else b=$10; sum[$1]+=b } END{ for(i in sum) print sum[i] " bytes for " i }' access.log | sort -nr
```

---

## 2) Список ESTABLISHED TCP соединений
```bash
lsof -nP -iTCP -sTCP:ESTABLISHED
```

---

## 3) Анализ мониторинга (скриншот)

**Наблюдения:**
- Резкий рост числа соединений к базе данных, приближающийся к лимиту (`max_connections`).
- Параллельный пик активных потоков/запросов и рост количества запросов (Questions).
- Пик активности приходится примерно на интервал 13:00–13:10.

**Вывод:**
- Вероятная причина недоступности — исчерпание соединений к базе данных (connection saturation), что вызвало деградацию/недоступность бэкенда.
- Время недоступности оценочно: около 13:00–13:10.

**Рекомендации:**
- **Краткосрочно:** снизить нагрузку, перезапустить приложение/освободить соединения, перенаправить read-запросы на реплики.
- **Среднесрочно:** увеличить `max_connections` при возможности, внедрить connection pooling.
- **Долгосрочно:** оптимизировать медленные запросы, масштабировать БД, настроить оповещения при достижении 80–90% лимита.

---

## 4) Dockerfile — рекомендации

- Не использовать `ubuntu:latest`; зафиксировать тег (например `ubuntu:22.04`) или применять `nginx:stable-alpine`.
- Заменить `MAINTAINER` на `LABEL maintainer="..."`.
- Добавить `.dockerignore` и копировать только необходимые файлы.
- Объединять `apt-get update` и `apt-get install` в одном RUN и очищать кеш (`rm -rf /var/lib/apt/lists/*`).
- Добавить `EXPOSE` и `HEALTHCHECK`.

**Рекомендуемый (компактный) вариант на базе официального образа:**
```dockerfile
FROM nginx:stable-alpine
LABEL maintainer="ops@example.com"
COPY ./public /usr/share/nginx/html
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD wget -qO- http://localhost/ || exit 1
CMD ["nginx", "-g", "daemon off;"]
```

---

## 5) Быстрая диагностика Nginx

1. `systemctl status nginx`
2. `curl -sS -D- http://127.0.0.1:80/ -o /dev/null`
3. `journalctl -u nginx -n 200` и `/var/log/nginx/error.log`
4. `nginx -t`
5. `ss -ltnp | grep :80`
6. `df -h`, `df -i`, `ls -l /var/www/html`
7. `top`, `free -m`, `dmesg | tail`
8. `ufw status`, `sestatus`
9. `systemctl restart nginx` (если нужно)

---

## 6) Развёртывание в Kubernetes

**Deployment (пример):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app-image:latest
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

**Service (пример):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
```

**HPA (пример):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

---

## 7) Инцидент: «Проект не работает» — краткий план действий

1. Подтвердить инцидент и определить приоритет/ETА.
2. Собрать информацию (время, примеры ошибок, регион).
3. Проверить мониторинг и логи.
4. Проверить доступность ключевых компонентов (LB, веб, БД, кеши).
5. Быстрые меры восстановления: переключить трафик, откат релиза, рестарт сервисов.
6. Провести RCA и внедрить превентивные меры.

