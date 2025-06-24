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

3. **Выводит всех клиентов и их объекты охраны**

```sql
SELECT 
  c.company_name,
  so.name AS security_object_name,
  so.address
FROM clients c
LEFT JOIN security_objects so ON c.id = so.client_id
ORDER BY c.company_name;

```

*Показывает название охранной организации, клиента и город в котором они работают.*

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

## Хранимая процедура

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
DELIMITER $$

CREATE TRIGGER before_salary_update
BEFORE UPDATE ON employees
FOR EACH ROW
BEGIN
    DECLARE msg_text VARCHAR(255);
    
    IF NEW.salary > OLD.salary THEN
        SET msg_text = CONCAT('Зарплата сотрудника "', NEW.full_name, '" увеличена с ', OLD.salary, ' до ', NEW.salary);
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = msg_text;
    END IF;
END $$

DELIMITER ;

```

*Срабатывает перед обновлением строки в таблице `employees`. Сравнивает старую зарплату (OLD.salary) и новую (NEW.salary). Если зарплата увеличилась, выдает сообщение с помощью SIGNAL SQLSTATE '45000'*

---

## Функции

```sql
CREATE FUNCTION get_salary_level(salary DECIMAL(10,2))
RETURNS VARCHAR(20)
DETERMINISTIC
BEGIN
    DECLARE level VARCHAR(20);

    IF salary < 30000 THEN
        SET level = 'Низкий уровень';
    ELSEIF salary >= 30000 AND salary < 70000 THEN
        SET level = 'Средний уровень';
    ELSE
        SET level = 'Высокий уровень';
    END IF;

    RETURN level;
END
```
*Функция определяет "уровень зарплаты" сотрудника на основе её числового значения.*

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
