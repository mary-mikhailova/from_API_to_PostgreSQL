# Выполнение задания

- Каждый день происходит выгрузка с API за вчерашний день
- Данные валидируются и заливаются в локально развернутую БД
- Производится анализ с помощью PostgeSQL
- Результат выгружается в гугл таблицу 
- По завершении происходит автоматическая рассылка-уведомление на почту
- На каждом этапе настроено логирование ошибок и промежуточных результатов

### 1. Организация логирования
File Handler хранит только 3 архвных файла - остальные будем удалять. Примерный вид логов:
<img width="637" height="312" alt="image" src="https://github.com/user-attachments/assets/813f56cd-a083-4bd0-897d-2a794b432853" />

### 2. Выргрузка данных
Учитаваем часовой пояс - 3 часа, выгружаем данные по API с помощью функции `get_api_data(api_url, params)`

### 3. Валидация данных
- Распаковываем json строку `passback_params = json.loads(attempt['passback_params'].replace("'", "\""))`
- Убираем попытки пользователей, имеющие пустые поля (которые ожидаются непустыми)
- Конвертируем дату-строку в формат даты

### 4. Создание классса для подсключение и загрузки данных в БД
- Превариательно локально развернута БД
- Используем шаблон Singleton
- Создаем таблицу с помощью запроса:
```
  create_table_query = f"""
            CREATE TABLE IF NOT EXISTS {table_name} (
            id SERIAL PRIMARY KEY,
            user_id VARCHAR(255) NOT NULL,
            oauth_consumer_key VARCHAR(255),
            lis_result_sourcedid TEXT,
            lis_outcome_service_url TEXT,
            is_correct BOOLEAN,
            attempt_type VARCHAR(10),
            created_at TIMESTAMP NOT NULL
            );
            """
  ```
- загружаем данные в БД:
```
insert_query = f"""
                    INSERT INTO {tablename}(user_id, oauth_consumer_key, lis_result_sourcedid, 
                    lis_outcome_service_url, is_correct, attempt_type, created_at)
                    VALUES (%s, %s, %s, %s, %s, %s, %s)
                    ON CONFLICT (user_id, created_at) DO NOTHING;
                    """
```
### 5. Выполнение любой необходимой аналитики
Например:
```
query = """
            WITH all_submits AS (
                SELECT user_id, is_correct, attempt_type
                FROM grader g 
                WHERE attempt_type='submit'
            )
            SELECT 'Количество попыток (submit)' as "Показатель",
            count(user_id) as "Значение"
            FROM all_submits
            union all
            select 'Количество успешных попыток (submit)' as "Показатель", 
            count(case when is_correct=true then user_id end) as "Значение"
            from all_submits
            union all
            select 'Количество уникальных юзеров' as "Показатель", 
            count(distinct user_id) as "Значение"
            from grader;
            """
```
### 6. Загружаем результат анализа в Гугл таблицу 
- Используем функцию `upload_to_google_sheets(results, spreadsheet_name, worksheet_name)`
- Получим:
<img width="468" height="83" alt="image" src="https://github.com/user-attachments/assets/e3bfc04a-ff89-4eb1-a26b-4dd9a6ce7480" />

### 7. Отправим email о готовности отчета коллегам 
- Используем функцию `send_mail()`
- Для безопасности пароль от почтового ящика будем получать из переменной окружения `password = os.getenv("send_email_for_python_project")`
- Пример письма:
<img width="773" height="224" alt="image" src="https://github.com/user-attachments/assets/2afbd717-826e-45d4-b2cf-029ad164bd06" />

