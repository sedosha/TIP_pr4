# Седова М.А., ЭФМО-01-25
# Практическая работа 4 Маршрутизация с chi. Создание небольшого CRUD-сервиса «Список задач».
# Дата: 07.10.2025

## Цели практической:
- Освоить основы создания REST API на языке Go
- Изучить работу с роутером Chi
- Реализовать CRUD операции для сущности "Задача"
- Научиться работать с middleware
- Реализовать дополнительные функции: валидацию, пагинацию, фильтрацию

## Дерево проекта
<img width="447" height="442" alt="image" src="https://github.com/user-attachments/assets/9217c547-b833-4aeb-8f3f-d3b4b46bd0bb" />

## Ключевые фрагменты кода
- **Роутер и версионирование API (main.go)**:
```
func main() {
    repo := task.NewRepo()
    h := task.NewHandler(repo)

    r := chi.NewRouter()
    r.Use(chimw.RequestID)
    r.Use(chimw.Recoverer)
    r.Use(myMW.Logger)
    r.Use(myMW.SimpleCORS)

    // API версия v1
    r.Route("/api/v1", func(v1 chi.Router) {
        v1.Mount("/tasks", h.Routes())
    })

    // Legacy версия для обратной совместимости
    r.Route("/api", func(api chi.Router) {
        api.Mount("/tasks", h.Routes())
    })

    log.Fatal(http.ListenAndServe(":8080", r))
}
```

- **Репозиторий с сохранением в файл (repo.go)**:
```
func (r *Repo) GetWithPagination(page, limit int, done *bool) ([]*Task, int) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    // Фильтрация по статусу done
    allTasks := make([]*Task, 0, len(r.items))
    for _, t := range r.items {
        if done == nil || t.Done == *done {
            allTasks = append(allTasks, t)
        }
    }
    
    total := len(allTasks)
    
    // Пагинация
    start := (page - 1) * limit
    if start >= total {
        return []*Task{}, total
    }
    
    end := start + limit
    if end > total {
        end = total
    }
    
    return allTasks[start:end], total
}

func (r *Repo) saveToFile() {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    tasks := make([]*Task, 0, len(r.items))
    for _, t := range r.items {
        tasks = append(tasks, t)
    }
    
    data, _ := json.Marshal(tasks)
    os.WriteFile("tasks.json", data, 0644)
}
```

- **Middleware для логирования (logger.go)**:
```
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %s", r.Method, r.URL.Path, time.Since(start))
    })
}
```
## Примеры запросов и ответов
### До доп заданий
- **Проверка health**
<img width="1280" height="424" alt="image" src="https://github.com/user-attachments/assets/667e3b6a-a5d2-4aac-8c6e-e4f775524fd2" />

- **Создание задачи**
<img width="1280" height="560" alt="image" src="https://github.com/user-attachments/assets/9fdc5531-7b16-4875-85e9-f245e02add59" />

- **Получение списка задач**
<img width="1280" height="593" alt="image" src="https://github.com/user-attachments/assets/97fd1646-d680-4426-9308-6e96ac50e054" />

- **Получение задачи по ID**
<img width="1280" height="514" alt="image" src="https://github.com/user-attachments/assets/62964eb6-9afe-440c-8da2-4d3971de58d5" />

- **Обновление задачи**
<img width="1280" height="563" alt="image" src="https://github.com/user-attachments/assets/8f9a9f80-2b2c-4f02-8022-624c85cef7d2" />

- **Удаление задачи**
<img width="1280" height="428" alt="image" src="https://github.com/user-attachments/assets/90dcbd41-1dd2-4ab9-bcfb-a11f022e2624" />

### После доп заданий
- **Слишком короткий title**
<img width="1280" height="421" alt="image" src="https://github.com/user-attachments/assets/203fc81c-7be1-4a0e-ba6d-5933c8c2e01c" />

- **Слишком длинный title**
<img width="1280" height="451" alt="image" src="https://github.com/user-attachments/assets/ce449ffa-ac91-4e22-aa85-cb5f60813e63" />

- **Пустой title**
<img width="1280" height="437" alt="image" src="https://github.com/user-attachments/assets/aeec7bd4-41f7-4c57-b50f-f99728dc722d" />

- **Пагинация - первая страница**
<img width="1280" height="917" alt="image" src="https://github.com/user-attachments/assets/5f4bb3fe-774e-4ecf-869d-e23cf90a6bec" />

- **Пагинация - вторая страница**
<img width="1280" height="924" alt="image" src="https://github.com/user-attachments/assets/6f71a210-0484-4fca-921e-00cdeb031763" />

- **Фильтр - только выполненные задачи**
<img width="1280" height="892" alt="image" src="https://github.com/user-attachments/assets/4bdc1eaa-ea2c-49d7-a607-dea8e37100c7" />

- **Комбинация фильтра и пагинации**
<img width="1280" height="932" alt="image" src="https://github.com/user-attachments/assets/cdba1b79-7d1a-418d-b030-8d9841dc9aa3" />

- **Несуществующая задача**
<img width="1280" height="469" alt="image" src="https://github.com/user-attachments/assets/8bef3ae5-4b9f-4e66-a6a8-ee494104c13b" />

### Логирование
<img width="563" height="434" alt="image" src="https://github.com/user-attachments/assets/26e4cf32-1b90-4c60-a0ab-ee1538abd052" />

## Обработка ошибок и коды ответов
| HTTP код | Ситуация | Пример сообщения об ошибке |
|----------|----------|----------------------------|
| **200 OK** | Успешный запрос | - |
| **201 Created** | Успешное создание ресурса | - |
| **400 Bad Request** | Невалидные параметры запроса | `"invalid json"`, `"title is required"`, `"invalid id"` |
| **404 Not Found** | Ресурс не найден | `"task not found"` |
| **422 Unprocessable Entity** | Ошибки валидации данных | `"title must be at least 3 characters"`, `"title must not exceed 100 characters"` |

## Результаты тестирования
| Маршрут | Метод | Тестовый сценарий | Ожидаемый результат | Фактический результат |
|---------|--------|-------------------|---------------------|----------------------|
| `/health` | GET | Проверка доступности сервера | 200 OK |  200 OK |
| `/api/v1/tasks` | POST | Корректное создание задачи | 201 Created |  201 Created |
| `/api/v1/tasks` | POST | Title = "ab" (2 символа) | 422 ошибка валидации |  422 Unprocessable Entity |
| `/api/v1/tasks` | POST | Title = "" (пустой) | 400 Bad Request |  400 Bad Request |
| `/api/v1/tasks` | POST | Очень длинный title (>100 символов) | 422 ошибка валидации |  422 Unprocessable Entity |
| `/api/v1/tasks` | POST | Невалидный JSON | 400 Bad Request |  400 Bad Request |
| `/api/v1/tasks/abc` | GET | Невалидный ID | 400 Bad Request |  400 Bad Request |
| `/api/v1/tasks/99` | GET | Несуществующий ID | 404 Not Found |  404 Not Found |
| `/api/v1/tasks/1` | PUT | Обновление несуществующей задачи | 404 Not Found |  404 Not Found |
| `/api/v1/tasks/1` | DELETE | Удаление несуществующей задачи | 404 Not Found |  404 Not Found |
| `/api/v1/tasks?page=1&limit=2` | GET | Пагинация списка | 200 OK с данными |  200 OK |
| `/api/v1/tasks?done=true` | GET | Фильтр по статусу выполнения | 200 OK с отфильтрованными данными |  200 OK |
| `/api/v1/tasks?done=false&page=2&limit=3` | GET | Комбинация фильтра и пагинации | 200 OK с данными |  200 OK |

## Выводы
### Что получилось
- Работающее API которое запускается и отвечает на запросы
- Все CRUD операции работают: можно создавать, читать, обновлять и удалять задачи
- Валидация проверяет что title 3-100 символов
- Пагинация работает - можно задать page и limit
- Фильтр по статусу работает
- Данные сохраняются в файл tasks.json и не пропадают после перезапуска
- API версионирование - есть /api/v1/ и старый /api/
- Реализованы логирование запросов и CORS для кросс-доменных запросов

### Что было сложным
- Архитектура проекта - сложно было правильно разделить логику между слоями (handler, repository, model)
- Асинхронное сохранение - организация фонового сохранения данных в файл
- Работа с Chi router - настройка middleware цепочки и вложенных роутеров для версионирования 

### Что можно улучшить
- Добавить нормальную базу данных вместо файлов
- Написать тесты чтобы проверять что всё работает
- Добавить поиск по задачам кроме фильтрации


