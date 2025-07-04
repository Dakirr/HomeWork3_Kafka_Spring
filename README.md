*Кафки тут не будет, потому что в какой-то момент мне надоело дебажить не работавший конфиг.*

**Важно! Если запрос возвращает одно целое число - это ID объекта.**

# 0. Сервисы

- API service (http://localhost:8081/swagger-ui/index.html)
  - Api Controller: Перенаправляет запросы в Payments и Orders.
- Payments service
  - Payments Controller: Отвечает за работу с банковскими аккаунтами и оплату заказов.
  - Messaging Controller: Отвечает за асинхронную работу с заказами из очереди на оплату и работает с Broker Service.
- Orders service
  - Orders Controller: Отвечает за создание заказов.
  - Messaging Controller: Отвечает за асинхронную отправку заказов в Broker Service.
- Broker service
  - Broker Controller: Является очередью сообщений между Payments и Orders.
- Frontend service (http://localhost:8080/)
  - Немного фронтенда. Работают геттеры и посттеры из API.
  - Доступен на http://localhost:8080/
  - Должно выглядеть как-то так: https://github.com/user-attachments/assets/e4163f1c-f1f2-4a3c-86ad-384c7eb9e2f7

# 1. Принцип работы 

Опишем принцип оплаты заказа, потому что это самое интересное:
  1. Данные для нового заказа поступают в API Service.
     
  2. Данные пересылаются в Orders service.
     
  3. В датабазе Orders Service создается запись о новом заказе со статусом 0 (не обработан) и пометкой "актуален".
     
  4. Messaging Controller того же сервиса переодически берет запись из датабазы и с помощью одной транзакции:
     
    4.1. Меняет ее пометку на "не актуален"
    4.2. Отправляет ее в Broker Service.
    4.3. Ожидается HTTP статус от Broker Service.
      4.3.а. Если получен верный статус, операция заканчивается.
      4.3.b. Если получен статус об ошибке и/или сервис не доступен, статус проставляется равным 2 (ошибка), и работа с заказом завершается.
    
  5. В датабазе Broker Service создается запись о новом заказе со статусом 0 (не обработан) и stage = 0 (ожидает отправки в Payments Service).
      
  6. Контроллер переодически берет запись из датабазы и с помощью одной транзакции:
      
    6.1. Меняет ее stage на 1 (Ожидает ответа от Payments Service).
    6.2. Отправляет ее в Payments Service.
    6.3. Ожидается HTTP статус от Payments Service.
       6.3.a. Если получен верный статус, операция заканчивается.
       6.3.b. Если получен статус об ошибке и/или сервис не доступен, проставляются статус = 2 (ошибка) и stage = 2 (Ожидает отправки в Orders Service).
       
  7. В датабазе Payment Service создается запись о новом заказе со статусом 0 (не обработан) и пометкой "актуален".
      
  8. Контроллер переодически делает следующий набор действий:
      
    8.0. Проверяет, есть ли во внутренней очереди заказы.
    8.0.a. Если нет - берет запись со статусом 0 из датабазы и с помощью одной транзакции:
      8.1. Меняет ее stage на 2 (Ожидает отправки в Orders Service).
      8.2. Проверяет существование банковского аккаунта с искомым id и достаточность средств на балансе.
        8.2.a. Если одно из этих условий не выполнено, проставляет статус = 2.
        8.2.b. Если оба условия выполнены:
          8.2.b.1. Вычитает искомую сумму из баланса аккаунта.
          8.2.b.2. Проставляет статус = 1 (успех).
      8.3. Пытается отправить заказ в Broker Service и ждет статуса HTTP:
          8.3.a. Если получен верный статус, операция заканчивается.
          8.3.b. Если получен статус об ошибке и/или сервис не доступен: cтатус добавляется во внутреннюю очередь.
    8.0.b. Если во внутренней очереди есть записи, берет одни из них и повторяет для нее шаг 8.3.
  
  9. Broker Service принимает запрос от Payments Service и проставляет stage = 2 (Ожидает отправки в Orders Service).

  10. Контроллер периодически берет из датабазы заказ со статусом 2 и с помощью одной транзакции делает следующее:
    
    10.1. Меняет ее stage на 3 (Завершено). 
    10.2. Пытается отправить заказ в Orders Service и ждет статуса HTTP:
    10.2.a. Если получен верный статус, операция заканчивается.
    10.2.b. Если получен статус об ошибке и/или сервис не доступен, проставляются stage = 2 (Ожидает отправки в Orders Service).
    
  11. Orders Service синхронизирует статус в соответствии с пришедшим ему в теле запроса объектом.
