import os, sys, pymysql
from PyQt6.QtWidgets import *
from PyQt6.QtCore import Qt
from PyQt6.QtGui import QPixmap

# подключение к базе данных
DB = {'host': 'localhost', 'user': 'root', 'password': 'Tipelka.006', 'database': 'shop',
      'cursorclass': pymysql.cursors.DictCursor}


def db(): return pymysql.connect(**DB)


# класс авторизации
class Login(QDialog):
    def __init__(self):
        super().__init__()
        self.setWindowTitle('Вход')
        self.setModal(True)
        layout = QVBoxLayout(self)

        # поле для ввода логина
        self.user = QLineEdit()
        self.user.setPlaceholderText('Логин')

        # поле для ввода пароля
        self.passw = QLineEdit()
        self.passw.setPlaceholderText('Пароль')
        self.passw.setEchoMode(QLineEdit.EchoMode.Password)

        # кнопка входа
        btn = QPushButton('Войти')
        btn.clicked.connect(self.check)

        # кнопка входа как гость
        guest_btn = QPushButton('Войти как гость')
        guest_btn.clicked.connect(self.guest_check)

        layout.addWidget(self.user)
        layout.addWidget(self.passw)
        layout.addWidget(btn)
        layout.addWidget(guest_btn)

        self.user_id = self.username = self.role_id = None

    # проверка на гостя
    def guest_check(self):
        self.user_id, self.role_id, self.username = None, 3, 'Гость'
        self.accept()

    # проверка существования логина и пароля в бд
    def check(self):
        c = db()
        cur = c.cursor()
        cur.execute("select user_id, username, role_id from users where username=%s and password_hash=%s",
                    (self.user.text(), self.passw.text()))
        u = cur.fetchone()
        c.close()
        if not u:
            QMessageBox.warning(self, 'Ошибка', 'Неверный логин или пароль')
            return
        self.user_id, self.username, self.role_id = u['user_id'], u['username'], u['role_id']
        self.accept()


# главный класс администратора (содержит весь функционал)
class AdminWindow(QMainWindow):
    def __init__(self, username):
        super().__init__()
        self.username = username
        self.selected_product_id = None
        self.selected_card = None

        # настройка окна
        self.setWindowTitle(f'Магазин | Админ: {username}')
        self.resize(1200, 750)

        # создание центрального виджета
        central = QWidget()
        self.setCentralWidget(central)
        main_layout = QVBoxLayout(central)

        # панель фильтрации
        filter_panel = QHBoxLayout()

        # поле поиска
        self.search_input = QLineEdit()
        self.search_input.setPlaceholderText('Поиск...')
        self.search_input.textChanged.connect(self.load_products)
        filter_panel.addWidget(self.search_input)

        # выпадающий список для сортировки
        self.sort_combo = QComboBox()
        self.sort_combo.addItems(['По умолчанию', 'По остатку на складе +', 'По остатку на складе -'])
        self.sort_combo.currentTextChanged.connect(self.load_products)
        filter_panel.addWidget(QLabel('Сортировка:'))
        filter_panel.addWidget(self.sort_combo)

        # фильтр по поставщику
        self.supplier_filter = QComboBox()
        self.supplier_filter.addItem('Все поставщики')
        self.supplier_filter.currentTextChanged.connect(self.load_products)
        filter_panel.addWidget(QLabel('Поставщик:'))
        filter_panel.addWidget(self.supplier_filter)

        main_layout.addLayout(filter_panel)

        # кнопки действий
        actions_layout = QHBoxLayout()
        self.btn_add = QPushButton('+ Добавить товар')
        self.btn_add.clicked.connect(self.add_product)
        self.btn_edit = QPushButton('Редактировать')
        self.btn_edit.clicked.connect(self.edit_selected_product)
        self.btn_delete = QPushButton('Удалить')
        self.btn_delete.clicked.connect(self.delete_selected_product)
        actions_layout.addWidget(self.btn_add)
        actions_layout.addWidget(self.btn_edit)
        actions_layout.addWidget(self.btn_delete)
        main_layout.addLayout(actions_layout)

        # область прокрутки с товарами
        self.scroll = QScrollArea()
        self.scroll.setWidgetResizable(True)
        self.container = QWidget()
        self.grid = QVBoxLayout(self.container)
        self.scroll.setWidget(self.container)
        main_layout.addWidget(self.scroll)

        # загрузка данных
        self.load_suppliers()
        self.load_products()

    # загрузка поставщиков для фильтра
    def load_suppliers(self):
        c = db()
        cur = c.cursor()
        cur.execute("SELECT supplier_name FROM suppliers")
        for s in cur.fetchall():
            self.supplier_filter.addItem(s['supplier_name'])
        c.close()

    # загрузка справочников
    def get_data(self, table, id_col, name_col):
        c = db()
        cur = c.cursor()
        cur.execute(f"SELECT {id_col}, {name_col} FROM {table}")
        data = cur.fetchall()
        c.close()
        return data

    # обработка выбора карточки
    def select_card(self, pid, card):
        if self.selected_card and self.selected_card != card:
            self.selected_card.setStyleSheet("QFrame{background:white;border:1px solid #ddd;margin:5px;padding:10px;}")
        self.selected_product_id = pid
        self.selected_card = card
        self.selected_card.setStyleSheet("QFrame{background:#e3f2fd;border:2px solid #2196f3;margin:5px;padding:10px;}")

    # редактирование выбранного товара
    def edit_selected_product(self):
        if not self.selected_product_id:
            QMessageBox.warning(self, 'Ошибка', 'Выберите товар')
            return
        self.edit_product(self.selected_product_id)

    # удаление выбранного товара
    def delete_selected_product(self):
        if not self.selected_product_id:
            QMessageBox.warning(self, 'Ошибка', 'Выберите товар')
            return
        self.delete_product(self.selected_product_id)

    # загрузка товаров с фильтрацией
    def load_products(self):
        self.selected_product_id = self.selected_card = None
        while self.grid.count():
            w = self.grid.takeAt(0).widget()
            if w: w.deleteLater()

        c = db()
        cur = c.cursor()
        q = """SELECT p.product_id, p.product_name, p.price, p.discount, p.stock, p.image_path, p.description,
                      s.supplier_name as supplier, m.manufacturer_name as manufacturer,
                      c.category_name as category, u.unit_name as unit
               FROM products p
               LEFT JOIN suppliers s ON p.supplier_id=s.supplier_id
               LEFT JOIN manufacturers m ON p.manufacturer_id=m.manufacturer_id
               LEFT JOIN categories c ON p.category_id=c.category_id
               LEFT JOIN units u ON p.unit_id=u.unit_id WHERE 1=1"""
        params = []

        # поиск
        if t := self.search_input.text().strip():
            q += " AND p.product_name LIKE %s"
            params.append(f'%{t}%')

        # фильтр по поставщику
        if self.supplier_filter.currentText() != 'Все поставщики':
            q += " AND s.supplier_name=%s"
            params.append(self.supplier_filter.currentText())

        # сортировка
        if self.sort_combo.currentText() == 'По остатку на складе +':
            q += " ORDER BY p.stock DESC"
        elif self.sort_combo.currentText() == 'По остатку на складе -':
            q += " ORDER BY p.stock ASC"

        cur.execute(q, params)
        products = cur.fetchall()
        c.close()

        if not products:
            lbl = QLabel('Нет товаров')
            lbl.setStyleSheet("color:black;")
            lbl.setAlignment(Qt.AlignmentFlag.AlignCenter)
            self.grid.addWidget(lbl)
            return

        for p in products:
            self.create_card(p)
        self.grid.addStretch()

    # создание карточки товара
    def create_card(self, p):
        card = QFrame()
        card.setStyleSheet("QFrame{background:white;border:1px solid #ddd;margin:5px;padding:10px;}")
        layout = QHBoxLayout(card)

        # изображение
        img = QLabel()
        img.setFixedSize(150, 150)
        img.setAlignment(Qt.AlignmentFlag.AlignCenter)
        img.setStyleSheet("border:1px solid #ccc;background:#f9f9f9;color:black;")
        if p.get('image_path') and os.path.exists(p['image_path']):
            img.setPixmap(QPixmap(p['image_path']).scaled(130, 130, Qt.AspectRatioMode.KeepAspectRatio))
        else:
            img.setText('Нет фото')
        layout.addWidget(img)

        # информация
        info = QVBoxLayout()
        info.setSpacing(5)
        cat = p.get('category', 'Без категории')
        cat_name_label = QLabel(f'Категория: {cat} | Название: {p["product_name"]}')
        cat_name_label.setStyleSheet("color: black;")
        info.addWidget(cat_name_label)

        if p.get('description'):
            d = p['description'][:100] + ('...' if len(p['description']) > 100 else '')
            desc_label = QLabel(d)
            desc_label.setStyleSheet("color: #666; font-size: 12px;")
            info.addWidget(desc_label)

        man_label = QLabel(f'Производитель: {p.get("manufacturer", "-")}')
        man_label.setStyleSheet("color: black;")
        info.addWidget(man_label)

        sup_label = QLabel(f'Поставщик: {p.get("supplier", "-")}')
        sup_label.setStyleSheet("color: black;")
        info.addWidget(sup_label)

        # цена со скидкой
        price = float(p['price'])
        disc = p.get('discount', 0)
        final_price = price * (1 - disc / 100) if disc > 0 else price
        if disc > 0:
            hl = QHBoxLayout()
            old_price = QLabel(f'Старая цена: {price:.2f} руб')
            old_price.setStyleSheet("text-decoration: line-through; color: #999;")
            new_price = QLabel(f'Новая цена: {final_price:.2f} руб')
            new_price.setStyleSheet("color: #e53935;")
            hl.addWidget(old_price)
            hl.addWidget(new_price)
            info.addLayout(hl)
        else:
            price_label = QLabel(f'Цена: {price:.2f} руб')
            price_label.setStyleSheet("color: black;")
            info.addWidget(price_label)

        unit_label = QLabel(f'Единица измерения: {p.get("unit", "шт")}')
        unit_label.setStyleSheet("color: black;")
        info.addWidget(unit_label)

        stock_label = QLabel(f'Количество на складе: {p.get("stock", 0)} шт')
        stock_label.setStyleSheet("color: black;")
        info.addWidget(stock_label)

        layout.addLayout(info, stretch=1)

        # блок скидки
        right = QVBoxLayout()
        right.setAlignment(Qt.AlignmentFlag.AlignCenter)
        if disc > 0:
            color = 'red' if disc > 15 else 'green'
            discount_label = QLabel(f'-{disc}%')
            discount_label.setStyleSheet(f"color: {color}; font-size: 18px; font-weight: bold;")
            right.addWidget(discount_label)
        layout.addLayout(right)

        # клик для выбора
        card.mousePressEvent = lambda e, pid=p['product_id'], c=card: self.select_card(pid, c)
        self.grid.addWidget(card)

    # добавление товара
    def add_product(self):
        d = QDialog(self)
        d.setWindowTitle('Добавление товара')
        d.setModal(True)
        d.resize(500, 600)
        layout = QVBoxLayout(d)
        form = QFormLayout()

        # поля ввода
        name = QLineEdit()
        cat = QComboBox()
        for c in self.get_data('categories', 'category_id', 'category_name'):
            cat.addItem(c['category_name'], c['category_id'])

        man = QComboBox()
        man.addItem('Не указан', None)
        for m in self.get_data('manufacturers', 'manufacturer_id', 'manufacturer_name'):
            man.addItem(m['manufacturer_name'], m['manufacturer_id'])

        sup = QComboBox()
        sup.addItem('Не указан', None)
        for s in self.get_data('suppliers', 'supplier_id', 'supplier_name'):
            sup.addItem(s['supplier_name'], s['supplier_id'])

        unit = QComboBox()
        for u in self.get_data('units', 'unit_id', 'unit_name'):
            unit.addItem(u.get('unit_name', u['unit_name']), u['unit_id'])

        price = QLineEdit()
        disc = QLineEdit()
        stock = QLineEdit()
        desc = QTextEdit()
        desc.setMaximumHeight(100)

        for w, lbl in [(name, 'Название:*'), (cat, 'Категория:'), (man, 'Производитель:'), (sup, 'Поставщик:'),
                       (unit, 'Единица измерения:'), (price, 'Цена:'), (disc, 'Скидка (%):'), (stock, 'Количество:'),
                       (desc, 'Описание:')]:
            form.addRow(lbl, w)
        layout.addLayout(form)

        # кнопки
        btns = QHBoxLayout()
        save = QPushButton('Сохранить')
        cancel = QPushButton('Отмена')
        btns.addWidget(save)
        btns.addWidget(cancel)
        layout.addLayout(btns)

        # сохранение
        def do_save():
            if not name.text():
                QMessageBox.warning(d, 'Ошибка', 'Введите название')
                return
            try:
                c = db()
                cur = c.cursor()
                cur.execute("""INSERT INTO products (product_name, category_id, manufacturer_id, supplier_id, unit_id, price, discount, stock, description)
                               VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s)""",
                            (name.text(), cat.currentData(), man.currentData(), sup.currentData(),
                             unit.currentData(), float(price.text() or 0), int(disc.text() or 0),
                             int(stock.text() or 0), desc.toPlainText()))
                c.commit()
                c.close()
                QMessageBox.information(d, 'Успех', 'Товар добавлен')
                d.accept()
                self.load_products()
            except Exception as e:
                QMessageBox.critical(d, 'Ошибка', str(e))

        save.clicked.connect(do_save)
        cancel.clicked.connect(d.reject)
        d.exec()

    # редактирование товара
    def edit_product(self, pid):
        c = db()
        cur = c.cursor()
        cur.execute("SELECT * FROM products WHERE product_id=%s", (pid,))
        prod = cur.fetchone()
        c.close()
        if not prod: return

        d = QDialog(self)
        d.setWindowTitle('Редактирование товара')
        d.setModal(True)
        d.resize(500, 600)
        layout = QVBoxLayout(d)
        form = QFormLayout()

        # поля с текущими значениями
        name = QLineEdit(prod['product_name'])
        cat = QComboBox()
        for c in self.get_data('categories', 'category_id', 'category_name'):
            cat.addItem(c['category_name'], c['category_id'])
            if c['category_id'] == prod.get('category_id'):
                cat.setCurrentIndex(cat.count() - 1)

        man = QComboBox()
        man.addItem('Не указан', None)
        for m in self.get_data('manufacturers', 'manufacturer_id', 'manufacturer_name'):
            man.addItem(m['manufacturer_name'], m['manufacturer_id'])
            if m['manufacturer_id'] == prod.get('manufacturer_id'):
                man.setCurrentIndex(man.count() - 1)

        sup = QComboBox()
        sup.addItem('Не указан', None)
        for s in self.get_data('suppliers', 'supplier_id', 'supplier_name'):
            sup.addItem(s['supplier_name'], s['supplier_id'])
            if s['supplier_id'] == prod.get('supplier_id'):
                sup.setCurrentIndex(sup.count() - 1)

        unit = QComboBox()
        for u in self.get_data('units', 'unit_id', 'unit_name'):
            unit.addItem(u.get('unit_name', u['unit_name']), u['unit_id'])
            if u['unit_id'] == prod.get('unit_id'):
                unit.setCurrentIndex(unit.count() - 1)

        price = QLineEdit(str(prod['price']))
        disc = QLineEdit(str(prod['discount']))
        stock = QLineEdit(str(prod['stock']))
        desc = QTextEdit(prod['description'] or '')
        desc.setMaximumHeight(100)

        for w, lbl in [(name, 'Название:*'), (cat, 'Категория:'), (man, 'Производитель:'), (sup, 'Поставщик:'),
                       (unit, 'Единица измерения:'), (price, 'Цена:'), (disc, 'Скидка (%):'), (stock, 'Количество:'),
                       (desc, 'Описание:')]:
            form.addRow(lbl, w)
        layout.addLayout(form)

        # кнопки
        btns = QHBoxLayout()
        save = QPushButton('Сохранить')
        cancel = QPushButton('Отмена')
        btns.addWidget(save)
        btns.addWidget(cancel)
        layout.addLayout(btns)

        # обновление
        def do_save():
            if not name.text():
                QMessageBox.warning(d, 'Ошибка', 'Введите название')
                return
            try:
                c = db()
                cur = c.cursor()
                cur.execute("""UPDATE products SET product_name=%s, category_id=%s, manufacturer_id=%s, supplier_id=%s,
                               unit_id=%s, price=%s, discount=%s, stock=%s, description=%s WHERE product_id=%s""",
                            (name.text(), cat.currentData(), man.currentData(), sup.currentData(),
                             unit.currentData(), float(price.text() or 0), int(disc.text() or 0),
                             int(stock.text() or 0), desc.toPlainText(), pid))
                c.commit()
                c.close()
                QMessageBox.information(d, 'Успех', 'Товар обновлен')
                d.accept()
                self.load_products()
            except Exception as e:
                QMessageBox.critical(d, 'Ошибка', str(e))

        save.clicked.connect(do_save)
        cancel.clicked.connect(d.reject)
        d.exec()

    # удаление товара
    def delete_product(self, pid):
        if QMessageBox.question(self, 'Удаление', 'Удалить товар?',
                                QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No) == QMessageBox.StandardButton.Yes:
            try:
                c = db()
                cur = c.cursor()
                cur.execute("DELETE FROM order_items WHERE product_id=%s", (pid,))
                cur.execute("DELETE FROM products WHERE product_id=%s", (pid,))
                c.commit()
                c.close()
                QMessageBox.information(self, 'Успех', 'Товар удален')
                self.selected_product_id = self.selected_card = None
                self.load_products()
            except Exception as e:
                QMessageBox.critical(self, 'Ошибка', str(e))


# класс менеджера (только просмотр с фильтрами)
class ManagerWindow(AdminWindow):
    def __init__(self, username):
        super().__init__(username)
        self.setWindowTitle(f'Магазин | Менеджер: {username}')
        # удаляем кнопки добавления, редактирования и удаления
        for btn in self.findChildren(QPushButton):
            if btn.text() in ['+ Добавить товар', 'Редактировать', 'Удалить']:
                btn.deleteLater()


# класс клиента (только просмотр, без фильтров)
class ClientWindow(AdminWindow):
    def __init__(self, username):
        super().__init__(username)
        self.setWindowTitle(f'Магазин | Клиент: {username}')
        # удаляем кнопки
        for btn in self.findChildren(QPushButton):
            if btn.text() in ['+ Добавить товар', 'Редактировать', 'Удалить']:
                btn.deleteLater()
        # удаляем панель фильтрации
        self.search_input.deleteLater()
        self.sort_combo.deleteLater()
        self.supplier_filter.deleteLater()
        # удаляем надписи "Сортировка:" и "Поставщик:"
        for label in self.findChildren(QLabel):
            if label.text() in ['Сортировка:', 'Поставщик:']:
                label.deleteLater()
        self.search_input = self.sort_combo = self.supplier_filter = None


# класс гостя
class GuestWindow(ClientWindow):
    def __init__(self, username):
        super().__init__(username)
        self.setWindowTitle(f'Магазин | Гость: {username}')


# главная часть программы
if __name__ == "__main__":
    app = QApplication(sys.argv)
    login = Login()
    if login.exec() == QDialog.DialogCode.Accepted:
        if login.role_id == 3 and login.username == 'Гость':
            window = GuestWindow(login.username)
        elif login.role_id == 2:
            window = ManagerWindow(login.username)
        elif login.role_id == 1:
            window = AdminWindow(login.username)
        else:
            window = ClientWindow(login.username)
        window.show()
        sys.exit(app.exec())
        
        
        
        
        
        
        
-- MySQL Script generated by MySQL Workbench
-- Fri May 29 00:18:34 2026
-- Model: New Model    Version: 1.0
-- MySQL Workbench Forward Engineering

SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0;
SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0;
SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';

-- -----------------------------------------------------
-- Schema mydb
-- -----------------------------------------------------
-- -----------------------------------------------------
-- Schema shop
-- -----------------------------------------------------

-- -----------------------------------------------------
-- Schema shop
-- -----------------------------------------------------
CREATE SCHEMA IF NOT EXISTS `shop` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci ;
USE `shop` ;

-- -----------------------------------------------------
-- Table `shop`.`categories`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`categories` (
  `category_id` INT NOT NULL AUTO_INCREMENT,
  `category_name` VARCHAR(100) NOT NULL,
  `parent_category_id` INT NULL DEFAULT NULL,
  PRIMARY KEY (`category_id`),
  UNIQUE INDEX `category_name_UNIQUE` (`category_name` ASC) VISIBLE,
  INDEX `fk_categories_parent_idx` (`parent_category_id` ASC) VISIBLE,
  CONSTRAINT `fk_categories_parent`
    FOREIGN KEY (`parent_category_id`)
    REFERENCES `shop`.`categories` (`category_id`)
    ON DELETE SET NULL
    ON UPDATE CASCADE)
ENGINE = InnoDB
AUTO_INCREMENT = 16
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop`.`manufacturers`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`manufacturers` (
  `manufacturer_id` INT NOT NULL AUTO_INCREMENT,
  `manufacturer_name` VARCHAR(255) NOT NULL,
  PRIMARY KEY (`manufacturer_id`),
  UNIQUE INDEX `manufacturer_name_UNIQUE` (`manufacturer_name` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 11
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop`.`order_statuses`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`order_statuses` (
  `status_id` INT NOT NULL AUTO_INCREMENT,
  `status_name` VARCHAR(50) NOT NULL,
  PRIMARY KEY (`status_id`),
  UNIQUE INDEX `status_name_UNIQUE` (`status_name` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 6
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop`.`roles`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`roles` (
  `role_id` INT NOT NULL AUTO_INCREMENT,
  `role_name` VARCHAR(50) NOT NULL,
  PRIMARY KEY (`role_id`),
  UNIQUE INDEX `role_name_UNIQUE` (`role_name` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 4
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop`.`users`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`users` (
  `user_id` INT NOT NULL AUTO_INCREMENT,
  `username` VARCHAR(100) NOT NULL,
  `password_hash` VARCHAR(255) NOT NULL,
  `email` VARCHAR(255) NOT NULL,
  `role_id` INT NOT NULL,
  `created_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`user_id`),
  UNIQUE INDEX `username_UNIQUE` (`username` ASC) VISIBLE,
  UNIQUE INDEX `email_UNIQUE` (`email` ASC) VISIBLE,
  INDEX `fk_users_roles_idx` (`role_id` ASC) VISIBLE,
  CONSTRAINT `fk_users_roles`
    FOREIGN KEY (`role_id`)
    REFERENCES `shop`.`roles` (`role_id`)
    ON DELETE RESTRICT
    ON UPDATE CASCADE)
ENGINE = InnoDB
AUTO_INCREMENT = 6
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop`.`orders`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`orders` (
  `order_id` INT NOT NULL AUTO_INCREMENT,
  `user_id` INT NOT NULL,
  `order_date` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `status_id` INT NOT NULL,
  `total_amount` DECIMAL(10,2) NOT NULL DEFAULT '0.00',
  PRIMARY KEY (`order_id`),
  INDEX `fk_orders_users_idx` (`user_id` ASC) VISIBLE,
  INDEX `fk_orders_statuses_idx` (`status_id` ASC) VISIBLE,
  CONSTRAINT `fk_orders_statuses`
    FOREIGN KEY (`status_id`)
    REFERENCES `shop`.`order_statuses` (`status_id`)
    ON DELETE RESTRICT
    ON UPDATE CASCADE,
  CONSTRAINT `fk_orders_users`
    FOREIGN KEY (`user_id`)
    REFERENCES `shop`.`users` (`user_id`)
    ON DELETE RESTRICT
    ON UPDATE CASCADE)
ENGINE = InnoDB
AUTO_INCREMENT = 11
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop`.`suppliers`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`suppliers` (
  `supplier_id` INT NOT NULL AUTO_INCREMENT,
  `supplier_name` VARCHAR(255) NOT NULL,
  PRIMARY KEY (`supplier_id`),
  UNIQUE INDEX `supplier_name_UNIQUE` (`supplier_name` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 11
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop`.`units`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`units` (
  `unit_id` INT NOT NULL AUTO_INCREMENT,
  `unit_name` VARCHAR(20) NOT NULL,
  `unit_abbreviation` VARCHAR(10) NULL DEFAULT NULL,
  PRIMARY KEY (`unit_id`),
  UNIQUE INDEX `unit_name_UNIQUE` (`unit_name` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 11
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop`.`products`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`products` (
  `product_id` INT NOT NULL AUTO_INCREMENT,
  `product_name` VARCHAR(255) NOT NULL,
  `category_id` INT NULL DEFAULT NULL,
  `supplier_id` INT NULL DEFAULT NULL,
  `manufacturer_id` INT NULL DEFAULT NULL,
  `unit_id` INT NULL DEFAULT NULL,
  `price` DECIMAL(10,2) NOT NULL,
  `discount` INT NULL DEFAULT '0',
  `stock` INT NOT NULL DEFAULT '0',
  `image_path` VARCHAR(500) NULL DEFAULT NULL,
  `description` TEXT NULL DEFAULT NULL,
  PRIMARY KEY (`product_id`),
  INDEX `fk_products_categories_idx` (`category_id` ASC) VISIBLE,
  INDEX `fk_products_suppliers_idx` (`supplier_id` ASC) VISIBLE,
  INDEX `fk_products_manufacturers_idx` (`manufacturer_id` ASC) VISIBLE,
  INDEX `fk_products_units_idx` (`unit_id` ASC) VISIBLE,
  CONSTRAINT `fk_products_categories`
    FOREIGN KEY (`category_id`)
    REFERENCES `shop`.`categories` (`category_id`)
    ON DELETE SET NULL
    ON UPDATE CASCADE,
  CONSTRAINT `fk_products_manufacturers`
    FOREIGN KEY (`manufacturer_id`)
    REFERENCES `shop`.`manufacturers` (`manufacturer_id`)
    ON DELETE SET NULL
    ON UPDATE CASCADE,
  CONSTRAINT `fk_products_suppliers`
    FOREIGN KEY (`supplier_id`)
    REFERENCES `shop`.`suppliers` (`supplier_id`)
    ON DELETE SET NULL
    ON UPDATE CASCADE,
  CONSTRAINT `fk_products_units`
    FOREIGN KEY (`unit_id`)
    REFERENCES `shop`.`units` (`unit_id`)
    ON DELETE SET NULL
    ON UPDATE CASCADE)
ENGINE = InnoDB
AUTO_INCREMENT = 19
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop`.`order_items`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop`.`order_items` (
  `item_id` INT NOT NULL AUTO_INCREMENT,
  `order_id` INT NOT NULL,
  `product_id` INT NOT NULL,
  `quantity` INT NOT NULL,
  `price_per_item` DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (`item_id`),
  INDEX `fk_order_items_orders_idx` (`order_id` ASC) VISIBLE,
  INDEX `fk_order_items_products_idx` (`product_id` ASC) VISIBLE,
  CONSTRAINT `fk_order_items_orders`
    FOREIGN KEY (`order_id`)
    REFERENCES `shop`.`orders` (`order_id`)
    ON DELETE CASCADE
    ON UPDATE CASCADE,
  CONSTRAINT `fk_order_items_products`
    FOREIGN KEY (`product_id`)
    REFERENCES `shop`.`products` (`product_id`)
    ON DELETE RESTRICT
    ON UPDATE CASCADE)
ENGINE = InnoDB
AUTO_INCREMENT = 16
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


SET SQL_MODE=@OLD_SQL_MODE;
SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS;
SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS;
