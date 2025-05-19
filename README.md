# Курсовая chop_oxrana — база данных для частного охранного предприятия

![image](https://github.com/user-attachments/assets/b300f6d1-c5a3-4063-8d32-eb6165b73e9a)


## Типовые запросы

1. **Сотрудник с максимальной зарплатой**

```sql
SELECT e.id, e.full_name, p.title AS position, e.salary
FROM employees e
JOIN positions p ON p.id = e.position_id
ORDER BY e.salary DESC
LIMIT 1;
````
*Выводит сотрудника с самой высокой зарплатой.*

---

2. **Удалить сотрудника по полному имени**

```sql
DELETE FROM employees
WHERE id = (
  SELECT * FROM (
    SELECT id FROM employees
    WHERE full_name = 'Екатерина Владимировна Александровна'
  ) AS tmp
);
```

*Удаляет сотрудника по точному совпадению полного имени.*

---

3. **Отработанные часы сотрудника за период**

```sql
SELECT shift_date, TIMEDIFF(shift_end, shift_start) AS hours_worked
FROM shifts
WHERE employee_id = (
  SELECT id FROM employees WHERE full_name = 'Иван Иван Иванович'
)
AND shift_date BETWEEN '2024-04-01' AND '2024-04-15';
```

*Показывает количество часов, отработанных сотрудником за заданный период.*

---

4. **Количество сотрудников по должностям**

```sql
SELECT p.title, COUNT(*) AS employee_count
FROM employees e
JOIN positions p ON p.id = e.position_id
GROUP BY p.title;
```

*Выводит число сотрудников в каждой должности.*

---

5. **Средняя зарплата по должностям**

```sql
SELECT p.title, AVG(e.salary) AS avg_salary
FROM employees e
JOIN positions p ON p.id = e.position_id
GROUP BY p.title;
```

*Показывает среднюю зарплату по каждой должности.*

---

## Хранимые процедуры

### Процедура для добавления нового сотрудника с документами

```sql
CREATE PROCEDURE AddEmployee(
    IN p_full_name VARCHAR(150),
    IN p_position_id INT,
    IN p_phone VARCHAR(20),
    IN p_email VARCHAR(100),
    IN p_hire_date DATE,
    IN p_salary DECIMAL(10,2),
    IN p_passport VARCHAR(50),
    IN p_id_card VARCHAR(50),
    IN p_security_license VARCHAR(50),
    IN p_valid_until DATE
)
BEGIN
    INSERT INTO employees(full_name, position_id, phone, email, hire_date, salary)
    VALUES (p_full_name, p_position_id, p_phone, p_email, p_hire_date, p_salary);
    SET @emp_id = LAST_INSERT_ID();
    INSERT INTO employee_documents(employee_id, passport_number, id_card, security_license, valid_until)
    VALUES (@emp_id, p_passport, p_id_card, p_security_license, p_valid_until);
END //
```
*Процедура добавляет нового сотрудника с указанием всех основных данных и одновременно вставляет данные о документах сотрудника в связанную таблицу.*

---

### Пример вызова процедуры

```sql
CALL AddEmployee(
  'Егор Рожков Иванович',
  2,
  '+7 923 465 74 73',
  'egorroj@n0maci.ru',
  '2024-05-23',
  55000.00,
  '2818383232',
  '393689360719',
  '22342342333',
  '2026-12-31'
);
```

---

## Триггеры

```sql
CREATE TRIGGER before_employee_delete
BEFORE DELETE ON employees
FOR EACH ROW
BEGIN
  DELETE FROM employee_documents WHERE employee_id = OLD.id;
END;
```

*Триггер автоматически удаляет документы сотрудника при удалении записи из таблицы `employees`.*

---

## Функции

```sql
CREATE FUNCTION CheckINN(inn VARCHAR(12)) RETURNS BOOLEAN
BEGIN
  RETURN TRUE; -- или FALSE в зависимости от проверки
END;
```
*Функция проверяет корректность ИНН.*

---

## Представления

```sql
SELECT * FROM detail_employees;
```
*Представление для удобного просмотра информации по сотрудникам.*

---

## Роли

### Роль руководителя отдела

```sql
CREATE ROLE IF NOT EXISTS LeaderUser;
GRANT SELECT ON chop_oxrana.employees TO LeaderUser;
GRANT SELECT ON chop_oxrana.positions TO LeaderUser;
GRANT SELECT ON chop_oxrana.clients TO LeaderUser;
GRANT SELECT, INSERT ON chop_oxrana.shifts TO LeaderUser;
```
*Роль с правами на просмотр сотрудников и создание смен.*

---

```sql
CREATE ROLE IF NOT EXISTS Specialist;
GRANT SELECT, INSERT, UPDATE ON chop_oxrana.employees TO Specialist;
GRANT SELECT, INSERT, UPDATE ON chop_oxrana.employee_documents TO Specialist;
GRANT SELECT, INSERT, UPDATE ON chop_oxrana.positions TO Specialist;
GRANT SELECT ON chop_oxrana.shifts TO Specialist;
GRANT SELECT, INSERT ON chop_oxrana.payments TO Specialist;
GRANT EXECUTE ON FUNCTION chop_oxrana.CheckINN TO Specialist;
GRANT EXECUTE ON PROCEDURE chop_oxrana.AddEmployee TO Specialist;
GRANT EXECUTE ON PROCEDURE chop_oxrana.SalaryReport TO Specialist;
```
*Роль с расширенными правами управления сотрудниками, документами и выплатами.*
