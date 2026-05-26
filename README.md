import sys, pymysql, os
from PyQt6.QtWidgets import *
from PyQt6.QtCore import Qt
from PyQt6.QtGui import QPixmap

DB = {'host': 'localhost', 'user': 'root', 'password': 'Tipelka.006', 'database': 'shop_db_3nf',
      'cursorclass': pymysql.cursors.DictCursor}


def db(): return pymysql.connect(**DB)


class Login(QDialog):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Вход")
        self.setModal(True)
        layout = QVBoxLayout(self)

        self.user = QLineEdit()
        self.user.setPlaceholderText("Логин")

        self.passw = QLineEdit()
        self.passw.setPlaceholderText("Пароль")
        self.passw.setEchoMode(QLineEdit.EchoMode.Password)

        btn = QPushButton("Войти")
        btn.clicked.connect(self.check)

        guest_btn = QPushButton("Войти как гость")
        guest_btn.clicked.connect(self.guest_login)

        layout.addWidget(self.user)
        layout.addWidget(self.passw)
        layout.addWidget(btn)
        layout.addWidget(guest_btn)

        self.user_id = None
        self.username = None
        self.role_id = None

    def guest_login(self):
        self.role_id = 3  # Гость
        self.user_id = None
        self.username = "Гость"
        self.accept()

    def check(self):
        c = db()
        cur = c.cursor()
        cur.execute("SELECT user_id, username, role_id FROM users WHERE username=%s AND password_hash=%s",
                    (self.user.text(), self.passw.text()))
        u = cur.fetchone()
        c.close()

        if not u:
            QMessageBox.warning(self, "Ошибка", "Неверный логин или пароль")
            return

        self.role_id = u['role_id']
        self.user_id = u['user_id']
        self.username = u['username']

        self.accept()


class ProductDialog(QDialog):
    def __init__(self, product_data=None, parent=None):
        super().__init__(parent)
        self.product_data = product_data
        self.setWindowTitle("Добавление товара" if not product_data else "Редактирование товара")
        self.setModal(True)
        self.resize(400, 500)

        layout = QVBoxLayout(self)

        # Название товара
        layout.addWidget(QLabel("Название товара:"))
        self.name_edit = QLineEdit()
        layout.addWidget(self.name_edit)

        # Цена
        layout.addWidget(QLabel("Цена:"))
        self.price_edit = QLineEdit()
        layout.addWidget(self.price_edit)

        # Скидка
        layout.addWidget(QLabel("Скидка (%):"))
        self.discount_edit = QLineEdit()
        layout.addWidget(self.discount_edit)

        # Количество
        layout.addWidget(QLabel("Количество в наличии:"))
        self.stock_edit = QLineEdit()
        layout.addWidget(self.stock_edit)

        # Единица измерения
        layout.addWidget(QLabel("Единица измерения:"))
        self.unit_edit = QLineEdit()
        layout.addWidget(self.unit_edit)

        # Поставщик
        layout.addWidget(QLabel("Поставщик:"))
        self.supplier_combo = QComboBox()
        self.load_suppliers()
        layout.addWidget(self.supplier_combo)

        # Производитель
        layout.addWidget(QLabel("Производитель:"))
        self.manufacturer_combo = QComboBox()
        self.load_manufacturers()
        layout.addWidget(self.manufacturer_combo)

        # Описание
        layout.addWidget(QLabel("Описание:"))
        self.description_edit = QTextEdit()
        layout.addWidget(self.description_edit)

        # Путь к изображению
        layout.addWidget(QLabel("Путь к изображению:"))
        image_layout = QHBoxLayout()
        self.image_edit = QLineEdit()
        image_layout.addWidget(self.image_edit)
        browse_btn = QPushButton("Обзор...")
        browse_btn.clicked.connect(self.browse_image)
        image_layout.addWidget(browse_btn)
        layout.addLayout(image_layout)

        # Кнопки
        buttons = QHBoxLayout()
        save_btn = QPushButton("Сохранить")
        save_btn.clicked.connect(self.accept)
        cancel_btn = QPushButton("Отмена")
        cancel_btn.clicked.connect(self.reject)
        buttons.addWidget(save_btn)
        buttons.addWidget(cancel_btn)
        layout.addLayout(buttons)

        if product_data:
            self.load_product_data()

    def load_suppliers(self):
        try:
            c = db()
            cur = c.cursor()
            cur.execute("SELECT supplier_id, supplier_name FROM suppliers")
            suppliers = cur.fetchall()
            c.close()
            for sup in suppliers:
                self.supplier_combo.addItem(sup['supplier_name'], sup['supplier_id'])
        except Exception as e:
            print(f"Ошибка загрузки поставщиков: {e}")

    def load_manufacturers(self):
        try:
            c = db()
            cur = c.cursor()
            cur.execute("SELECT manufacturer_id, manufacturer_name FROM manufacturers")
            manufacturers = cur.fetchall()
            c.close()
            for man in manufacturers:
                self.manufacturer_combo.addItem(man['manufacturer_name'], man['manufacturer_id'])
        except Exception as e:
            print(f"Ошибка загрузки производителей: {e}")

    def browse_image(self):
        file_path, _ = QFileDialog.getOpenFileName(self, "Выберите изображение", "",
                                                   "Image Files (*.png *.jpg *.jpeg *.bmp)")
        if file_path:
            self.image_edit.setText(file_path)

    def load_product_data(self):
        self.name_edit.setText(self.product_data.get('name', ''))
        self.price_edit.setText(str(self.product_data.get('price', '')))
        self.discount_edit.setText(str(self.product_data.get('discount', 0)))
        self.stock_edit.setText(str(self.product_data.get('stock', 0)))
        self.unit_edit.setText(self.product_data.get('unit', 'шт'))
        self.description_edit.setText(self.product_data.get('description', ''))
        self.image_edit.setText(self.product_data.get('image_path', ''))

        # Выбираем поставщика
        supplier_name = self.product_data.get('supplier')
        if supplier_name:
            index = self.supplier_combo.findText(supplier_name)
            if index >= 0:
                self.supplier_combo.setCurrentIndex(index)

        # Выбираем производителя
        manufacturer_name = self.product_data.get('manufacturer')
        if manufacturer_name:
            index = self.manufacturer_combo.findText(manufacturer_name)
            if index >= 0:
                self.manufacturer_combo.setCurrentIndex(index)

    def get_product_data(self):
        return {
            'name': self.name_edit.text(),
            'price': float(self.price_edit.text()) if self.price_edit.text() else 0,
            'discount': int(self.discount_edit.text()) if self.discount_edit.text() else 0,
            'stock': int(self.stock_edit.text()) if self.stock_edit.text() else 0,
            'unit': self.unit_edit.text(),
            'description': self.description_edit.toPlainText(),
            'image_path': self.image_edit.text(),
            'supplier_id': self.supplier_combo.currentData(),
            'manufacturer_id': self.manufacturer_combo.currentData()
        }


class ClientWindow(QMainWindow):
    def __init__(s, username):
        super().__init__()
        s.username = username
        s.setWindowTitle(f"Магазин (Пользователь: {username})")
        s.resize(1200, 750)
        w = QWidget()
        s.setCentralWidget(w)
        l = QVBoxLayout(w)

        c = db()
        cur = c.cursor()
        cur.execute("SELECT user_id FROM users WHERE username=%s", (username,))
        result = cur.fetchone()
        s.user_id = result['user_id'] if result else None
        c.close()

        s.scroll = QScrollArea()
        s.scroll.setWidgetResizable(True)
        w2 = QWidget()

        s.grid = QVBoxLayout(w2)
        s.scroll.setWidget(w2)
        l.addWidget(s.scroll)
        s.load()

    def add_to_cart(s, product_id, product_name, price):
        try:
            c = db()
            cur = c.cursor()
            cur.execute("INSERT INTO orders (user_id, status_id, total_amount) VALUES (%s, 1, %s)",
                        (s.user_id, float(price)))
            order_id = cur.lastrowid

            cur.execute(
                "INSERT INTO order_items (order_id, product_id, quantity, price_per_item) VALUES (%s, %s, 1, %s)",
                (order_id, product_id, float(price)))
            c.commit()
            c.close()
            QMessageBox.information(s, "Добавлено!", f"Товар '{product_name}' добавлен в заказ №{order_id}")
        except Exception as e:
            QMessageBox.warning(s, "Ошибка", f"Не удалось добавить товар: {e}")

    def load(s):
        while s.grid.count():
            i = s.grid.takeAt(0)
            if i.widget(): i.widget().deleteLater()

        c = db()
        cur = c.cursor()

        query = """
            SELECT 
                p.product_id,
                p.product_name as name,
                p.price,
                p.discount,
                p.stock,
                p.unit,
                p.image_path,
                p.description,
                s.supplier_name as supplier,
                m.manufacturer_name as manufacturer
            FROM products p
            LEFT JOIN suppliers s ON p.supplier_id = s.supplier_id
            LEFT JOIN manufacturers m ON p.manufacturer_id = m.manufacturer_id
            WHERE 1=1
        """
        params = []

        cur.execute(query, params)
        products = cur.fetchall()
        c.close()

        for p in products:
            card = QFrame()
            card.setStyleSheet("background:white; border:1px solid #ddd; padding:10px; ")
            layout = QHBoxLayout(card)

            img = QLabel()
            img.setFixedSize(100, 100)
            img.setAlignment(Qt.AlignmentFlag.AlignCenter)
            img.setStyleSheet("background:#f0f0f0; border:1px solid #ccc; color: black;")

            if p.get('image_path') and os.path.exists(p['image_path']):
                pm = QPixmap(p['image_path'])
                pm = pm.scaled(100, 100, Qt.AspectRatioMode.KeepAspectRatio)
                img.setPixmap(pm)
            else:
                img.setText("НЕТ ФОТО")
            layout.addWidget(img)

            info = QVBoxLayout()
            info.addWidget(QLabel(f"<b style='color:black;'> {p['name']} </b>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Поставщик: {p.get('supplier', '-')} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Производитель: {p.get('manufacturer', '-')} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> В наличии: {p.get('stock', 0)} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Ед.:{p.get('unit', 'шт')}. </span>"))
            layout.addLayout(info)

            right = QVBoxLayout()
            discount = p.get('discount', 0)
            price = float(p['price'])
            final_price = price * (1 - discount / 100) if discount > 0 else price

            if discount > 15:
                right.addWidget(QLabel(f"<b style='background:#dc3545; color:white;'> -{discount}% </b>"))
                right.addWidget(QLabel(f"<s style='color:black;'>{price:.2f} ₽</s>"))
                right.addWidget(QLabel(f"<b style='color:red;'>{final_price:.2f} ₽</b>"))
            elif discount > 0:
                right.addWidget(QLabel(f"<b style='background:#2E8B57; color:white;'>-{discount}%</b>"))
                right.addWidget(QLabel(f"<s style='color:#2E8B57;'>{price:.2f} ₽</s>"))
                right.addWidget(QLabel(f"<b style='color:#2E8B57;'>{final_price:.2f} ₽</b>"))
            else:
                right.addWidget(QLabel(f"<b style='color:green;'>{price:.2f} ₽</b>"))

            if p.get('stock', 0) > 0:
                btn = QPushButton("🛒 В корзину")
                btn.setStyleSheet('background: #28a745; color: white;')
                btn.clicked.connect(lambda ch, pid=p['product_id'], name=p['name'], fprice=final_price:
                                    s.add_to_cart(pid, name, fprice))
                right.addWidget(btn)
            else:
                right.addWidget(QLabel("<b style='color:red;'> Нет в наличии</b>"))

            layout.addLayout(right)
            s.grid.addWidget(card)
        s.grid.addStretch()


class ManagerWindow(QMainWindow):
    def __init__(s, username):
        super().__init__()
        s.username = username
        s.setWindowTitle(f"Магазин - Менеджер ({username})")
        s.resize(1200, 750)
        w = QWidget()
        s.setCentralWidget(w)
        l = QVBoxLayout(w)

        c = db()
        cur = c.cursor()
        cur.execute("SELECT user_id FROM users WHERE username=%s", (username,))
        result = cur.fetchone()
        s.user_id = result['user_id'] if result else None
        c.close()

        # Панель фильтров как у админа
        filter_panel = QHBoxLayout()
        s.srch = QLineEdit()
        s.srch.setPlaceholderText(" Поиск...")
        s.srch.textChanged.connect(s.load)
        filter_panel.addWidget(s.srch)

        s.sort_combo = QComboBox()
        s.sort_combo.addItems(["По умолчанию", "По количеству ↑", "По количеству ↓"])
        s.sort_combo.currentTextChanged.connect(s.load)
        filter_panel.addWidget(QLabel("Сортировка:"))
        filter_panel.addWidget(s.sort_combo)

        s.supplier_filter_combo = QComboBox()
        s.supplier_filter_combo.addItem("Все поставщики")
        s.supplier_filter_combo.currentTextChanged.connect(s.load)
        s.load_suppliers()
        filter_panel.addWidget(QLabel("Поставщик:"))
        filter_panel.addWidget(s.supplier_filter_combo)

        l.addLayout(filter_panel)

        s.scroll = QScrollArea()
        s.scroll.setWidgetResizable(True)
        w2 = QWidget()
        s.grid = QVBoxLayout(w2)
        s.scroll.setWidget(w2)
        l.addWidget(s.scroll)
        s.load()

    def load_suppliers(s):
        try:
            c = db()
            cur = c.cursor()
            cur.execute("SELECT DISTINCT supplier_name as supplier FROM suppliers")
            suppliers = cur.fetchall()
            c.close()
            for sup in suppliers:
                if sup['supplier']:
                    s.supplier_filter_combo.addItem(sup['supplier'])
        except Exception as e:
            print(f"Ошибка загрузки поставщиков: {e}")

    def load(s):
        while s.grid.count():
            i = s.grid.takeAt(0)
            if i.widget(): i.widget().deleteLater()

        c = db()
        cur = c.cursor()
        t = s.srch.text().strip()

        query = """
            SELECT 
                p.product_id,
                p.product_name as name,
                p.price,
                p.discount,
                p.stock,
                p.unit,
                p.image_path,
                p.description,
                s.supplier_name as supplier,
                m.manufacturer_name as manufacturer
            FROM products p
            LEFT JOIN suppliers s ON p.supplier_id = s.supplier_id
            LEFT JOIN manufacturers m ON p.manufacturer_id = m.manufacturer_id
            WHERE 1=1
        """
        params = []

        if t:
            query += " AND p.product_name LIKE %s"
            params.append(f'%{t}%')

        supplier_filter = s.supplier_filter_combo.currentText()
        if supplier_filter != "Все поставщики":
            query += " AND s.supplier_name = %s"
            params.append(supplier_filter)

        sort_text = s.sort_combo.currentText()
        if sort_text == "По количеству ↑":
            query += " ORDER BY p.stock ASC"
        elif sort_text == "По количеству ↓":
            query += " ORDER BY p.stock DESC"

        cur.execute(query, params)
        products = cur.fetchall()
        c.close()

        if not products:
            lbl = QLabel("Товары не найдены")
            lbl.setAlignment(Qt.AlignmentFlag.AlignCenter)
            lbl.setStyleSheet("color: black;")
            s.grid.addWidget(lbl)
            return

        for p in products:
            card = QFrame()
            card.setStyleSheet("background:white; border:1px solid #ddd; border-radius:10px; padding:10px; margin:5px;")
            layout = QHBoxLayout(card)

            img = QLabel()
            img.setFixedSize(100, 100)
            img.setAlignment(Qt.AlignmentFlag.AlignCenter)
            img.setStyleSheet("background:#f0f0f0; border:1px solid #ccc; color: black;")

            if p.get('image_path') and os.path.exists(p['image_path']):
                pm = QPixmap(p['image_path'])
                pm = pm.scaled(100, 100, Qt.AspectRatioMode.KeepAspectRatio)
                img.setPixmap(pm)
            else:
                img.setText("НЕТ ФОТО")
            layout.addWidget(img)

            info = QVBoxLayout()
            info.addWidget(QLabel(f"<b style='color:black;'> {p['name']} </b>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Поставщик: {p.get('supplier', '-')} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Производитель: {p.get('manufacturer', '-')} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> В наличии: {p.get('stock', 0)} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Ед.:{p.get('unit', 'шт')}. </span>"))
            layout.addLayout(info)

            right = QVBoxLayout()
            discount = p.get('discount', 0)
            price = float(p['price'])
            final_price = price * (1 - discount / 100) if discount > 0 else price

            if discount > 15:
                right.addWidget(QLabel(f"<b style='background:#dc3545; color:white;'> -{discount}% </b>"))
                right.addWidget(QLabel(f"<s style='color:black;'>{price:.2f} ₽</s>"))
                right.addWidget(QLabel(f"<b style='color:red;'>{final_price:.2f} ₽</b>"))
            elif discount > 0:
                right.addWidget(QLabel(f"<b style='background:#2E8B57; color:white;'>-{discount}%</b>"))
                right.addWidget(QLabel(f"<s style='color:#2E8B57;'>{price:.2f} ₽</s>"))
                right.addWidget(QLabel(f"<b style='color:#2E8B57;'>{final_price:.2f} ₽</b>"))
            else:
                right.addWidget(QLabel(f"<b style='color:green;'>{price:.2f} ₽</b>"))

            layout.addLayout(right)
            s.grid.addWidget(card)
        s.grid.addStretch()


class AdminWindow(QMainWindow):
    def __init__(s, username):
        super().__init__()
        s.username = username
        s.setWindowTitle(f"Магазин - Администратор ({username})")
        s.resize(1200, 750)
        w = QWidget()
        s.setCentralWidget(w)
        l = QVBoxLayout(w)

        c = db()
        cur = c.cursor()
        cur.execute("SELECT user_id FROM users WHERE username=%s", (username,))
        result = cur.fetchone()
        s.user_id = result['user_id'] if result else None
        c.close()

        filter_panel = QHBoxLayout()
        s.srch = QLineEdit()
        s.srch.setPlaceholderText(" Поиск...")
        s.srch.textChanged.connect(s.load)
        filter_panel.addWidget(s.srch)

        s.sort_combo = QComboBox()
        s.sort_combo.addItems(["По умолчанию", "По количеству ↑", "По количеству ↓"])
        s.sort_combo.currentTextChanged.connect(s.load)
        filter_panel.addWidget(QLabel("Сортировка:"))
        filter_panel.addWidget(s.sort_combo)

        s.supplier_filter_combo = QComboBox()
        s.supplier_filter_combo.addItem("Все поставщики")
        s.supplier_filter_combo.currentTextChanged.connect(s.load)
        s.load_suppliers()
        filter_panel.addWidget(QLabel("Поставщик:"))
        filter_panel.addWidget(s.supplier_filter_combo)

        btn_add = QPushButton("➕ Новый товар")
        btn_add.setStyleSheet('background: #28a745; color: white;')
        btn_add.clicked.connect(s.add_product)
        filter_panel.addWidget(btn_add)

        l.addLayout(filter_panel)

        s.scroll = QScrollArea()
        s.scroll.setWidgetResizable(True)
        w2 = QWidget()
        s.grid = QVBoxLayout(w2)
        s.scroll.setWidget(w2)
        l.addWidget(s.scroll)
        s.load()

    def load_suppliers(s):
        try:
            c = db()
            cur = c.cursor()
            cur.execute("SELECT DISTINCT supplier_name as supplier FROM suppliers")
            suppliers = cur.fetchall()
            c.close()
            for sup in suppliers:
                if sup['supplier']:
                    s.supplier_filter_combo.addItem(sup['supplier'])
        except Exception as e:
            print(f"Ошибка загрузки поставщиков: {e}")

    def add_product(s):
        dialog = ProductDialog(parent=s)
        if dialog.exec() == QDialog.DialogCode.Accepted:
            data = dialog.get_product_data()
            try:
                c = db()
                cur = c.cursor()
                cur.execute("""
                    INSERT INTO products (product_name, price, discount, stock, unit, description, image_path, supplier_id, manufacturer_id)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
                """, (data['name'], data['price'], data['discount'], data['stock'], data['unit'],
                      data['description'], data['image_path'], data['supplier_id'], data['manufacturer_id']))
                c.commit()
                c.close()
                QMessageBox.information(s, "Успех", "Товар успешно добавлен!")
                s.load()
            except Exception as e:
                QMessageBox.warning(s, "Ошибка", f"Не удалось добавить товар: {e}")

    def edit_product(s, product_id, product_data):
        dialog = ProductDialog(product_data, parent=s)
        if dialog.exec() == QDialog.DialogCode.Accepted:
            data = dialog.get_product_data()
            try:
                c = db()
                cur = c.cursor()
                cur.execute("""
                    UPDATE products 
                    SET product_name=%s, price=%s, discount=%s, stock=%s, unit=%s, 
                        description=%s, image_path=%s, supplier_id=%s, manufacturer_id=%s
                    WHERE product_id=%s
                """, (data['name'], data['price'], data['discount'], data['stock'], data['unit'],
                      data['description'], data['image_path'], data['supplier_id'], data['manufacturer_id'],
                      product_id))
                c.commit()
                c.close()
                QMessageBox.information(s, "Успех", "Товар успешно обновлен!")
                s.load()
            except Exception as e:
                QMessageBox.warning(s, "Ошибка", f"Не удалось обновить товар: {e}")

    def delete_product(s, product_id, product_name):
        reply = QMessageBox.question(s, "Подтверждение",
                                     f"Вы уверены, что хотите удалить товар '{product_name}'?",
                                     QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No)
        if reply == QMessageBox.StandardButton.Yes:
            try:
                c = db()
                cur = c.cursor()
                cur.execute("DELETE FROM products WHERE product_id=%s", (product_id,))
                c.commit()
                c.close()
                QMessageBox.information(s, "Успех", "Товар успешно удален!")
                s.load()
            except Exception as e:
                QMessageBox.warning(s, "Ошибка", f"Не удалось удалить товар: {e}")

    def load(s):
        while s.grid.count():
            i = s.grid.takeAt(0)
            if i.widget(): i.widget().deleteLater()

        c = db()
        cur = c.cursor()
        t = s.srch.text().strip()

        query = """
            SELECT 
                p.product_id,
                p.product_name as name,
                p.price,
                p.discount,
                p.stock,
                p.unit,
                p.image_path,
                p.description,
                s.supplier_name as supplier,
                m.manufacturer_name as manufacturer
            FROM products p
            LEFT JOIN suppliers s ON p.supplier_id = s.supplier_id
            LEFT JOIN manufacturers m ON p.manufacturer_id = m.manufacturer_id
            WHERE 1=1
        """
        params = []

        if t:
            query += " AND p.product_name LIKE %s"
            params.append(f'%{t}%')

        supplier_filter = s.supplier_filter_combo.currentText()
        if supplier_filter != "Все поставщики":
            query += " AND s.supplier_name = %s"
            params.append(supplier_filter)

        sort_text = s.sort_combo.currentText()
        if sort_text == "По количеству ↑":
            query += " ORDER BY p.stock ASC"
        elif sort_text == "По количеству ↓":
            query += " ORDER BY p.stock DESC"

        cur.execute(query, params)
        products = cur.fetchall()
        c.close()

        if not products:
            lbl = QLabel("Товары не найдены")
            lbl.setAlignment(Qt.AlignmentFlag.AlignCenter)
            lbl.setStyleSheet("color: black;")
            s.grid.addWidget(lbl)
            return

        for p in products:
            card = QFrame()
            card.setStyleSheet("background:white; border:1px solid #ddd; border-radius:10px; padding:10px; margin:5px;")
            layout = QHBoxLayout(card)

            img = QLabel()
            img.setFixedSize(100, 100)
            img.setAlignment(Qt.AlignmentFlag.AlignCenter)
            img.setStyleSheet("background:#f0f0f0; border:1px solid #ccc; color: black;")

            if p.get('image_path') and os.path.exists(p['image_path']):
                pm = QPixmap(p['image_path'])
                pm = pm.scaled(100, 100, Qt.AspectRatioMode.KeepAspectRatio)
                img.setPixmap(pm)
            else:
                img.setText("НЕТ ФОТО")
            layout.addWidget(img)

            info = QVBoxLayout()
            info.addWidget(QLabel(f"<b style='color:black;'> {p['name']} </b>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Поставщик: {p.get('supplier', '-')} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Производитель: {p.get('manufacturer', '-')} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> В наличии: {p.get('stock', 0)} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Ед.:{p.get('unit', 'шт')}. </span>"))
            layout.addLayout(info)

            right = QVBoxLayout()
            discount = p.get('discount', 0)
            price = float(p['price'])
            final_price = price * (1 - discount / 100) if discount > 0 else price

            if discount > 15:
                right.addWidget(QLabel(f"<b style='background:#dc3545; color:white;'> -{discount}% </b>"))
                right.addWidget(QLabel(f"<s style='color:black;'>{price:.2f} ₽</s>"))
                right.addWidget(QLabel(f"<b style='color:red;'>{final_price:.2f} ₽</b>"))
            elif discount > 0:
                right.addWidget(QLabel(f"<b style='background:#2E8B57; color:white;'>-{discount}%</b>"))
                right.addWidget(QLabel(f"<s style='color:#2E8B57;'>{price:.2f} ₽</s>"))
                right.addWidget(QLabel(f"<b style='color:#2E8B57;'>{final_price:.2f} ₽</b>"))
            else:
                right.addWidget(QLabel(f"<b style='color:green;'>{price:.2f} ₽</b>"))

            btn_edit = QPushButton("✏️ Редактировать")
            btn_edit.setStyleSheet('background: #007bff; color: white;')
            btn_edit.clicked.connect(lambda ch, pid=p['product_id'], data=p: s.edit_product(pid, data))
            right.addWidget(btn_edit)

            btn_delete = QPushButton("🗑️ Удалить")
            btn_delete.setStyleSheet('background: #dc3545; color: white;')
            btn_delete.clicked.connect(lambda ch, pid=p['product_id'], name=p['name']: s.delete_product(pid, name))
            right.addWidget(btn_delete)

            layout.addLayout(right)
            s.grid.addWidget(card)
        s.grid.addStretch()


class GuestWindow(QMainWindow):
    def __init__(s, username):
        super().__init__()
        s.username = username
        s.setWindowTitle(f"Магазин (Гость: {username})")
        s.resize(1200, 750)
        w = QWidget()
        s.setCentralWidget(w)
        l = QVBoxLayout(w)

        s.scroll = QScrollArea()
        s.scroll.setWidgetResizable(True)
        w2 = QWidget()
        s.grid = QVBoxLayout(w2)
        s.scroll.setWidget(w2)
        l.addWidget(s.scroll)
        s.load()

    def load(s):
        while s.grid.count():
            i = s.grid.takeAt(0)
            if i.widget(): i.widget().deleteLater()

        c = db()
        cur = c.cursor()

        query = """
            SELECT 
                p.product_id,
                p.product_name as name,
                p.price,
                p.discount,
                p.stock,
                p.unit,
                p.image_path,
                p.description,
                s.supplier_name as supplier,
                m.manufacturer_name as manufacturer
            FROM products p
            LEFT JOIN suppliers s ON p.supplier_id = s.supplier_id
            LEFT JOIN manufacturers m ON p.manufacturer_id = m.manufacturer_id
            WHERE 1=1
        """
        params = []

        cur.execute(query, params)
        products = cur.fetchall()
        c.close()

        if not products:
            lbl = QLabel("Товары не найдены")
            lbl.setAlignment(Qt.AlignmentFlag.AlignCenter)
            lbl.setStyleSheet("color: black;")
            s.grid.addWidget(lbl)
            return

        for p in products:
            card = QFrame()
            card.setStyleSheet("background:white; border:1px solid #ddd; border-radius:10px; padding:10px; margin:5px;")
            layout = QHBoxLayout(card)

            img = QLabel()
            img.setFixedSize(100, 100)
            img.setAlignment(Qt.AlignmentFlag.AlignCenter)
            img.setStyleSheet("background:#f0f0f0; border:1px solid #ccc; color: black;")

            if p.get('image_path') and os.path.exists(p['image_path']):
                pm = QPixmap(p['image_path'])
                pm = pm.scaled(100, 100, Qt.AspectRatioMode.KeepAspectRatio)
                img.setPixmap(pm)
            else:
                img.setText("НЕТ ФОТО")
            layout.addWidget(img)

            info = QVBoxLayout()
            info.addWidget(QLabel(f"<b style='color:black;'> {p['name']} </b>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Поставщик: {p.get('supplier', '-')} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Производитель: {p.get('manufacturer', '-')} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> В наличии: {p.get('stock', 0)} </span>"))
            info.addWidget(QLabel(f"<span style='color:black;'> Ед.:{p.get('unit', 'шт')}. </span>"))
            layout.addLayout(info)

            right = QVBoxLayout()
            discount = p.get('discount', 0)
            price = float(p['price'])
            final_price = price * (1 - discount / 100) if discount > 0 else price

            if discount > 15:
                right.addWidget(QLabel(f"<b style='background:#dc3545; color:white;'> -{discount}% </b>"))
                right.addWidget(QLabel(f"<s style='color:black;'>{price:.2f} ₽</s>"))
                right.addWidget(QLabel(f"<b style='color:red;'>{final_price:.2f} ₽</b>"))
            elif discount > 0:
                right.addWidget(QLabel(f"<b style='background:#2E8B57; color:white;'>-{discount}%</b>"))
                right.addWidget(QLabel(f"<s style='color:#2E8B57;'>{price:.2f} ₽</s>"))
                right.addWidget(QLabel(f"<b style='color:#2E8B57;'>{final_price:.2f} ₽</b>"))
            else:
                right.addWidget(QLabel(f"<b style='color:green;'>{price:.2f} ₽</b>"))

            layout.addLayout(right)
            s.grid.addWidget(card)
        s.grid.addStretch()


if __name__ == "__main__":
    app = QApplication(sys.argv)
    login = Login()
    if login.exec() == QDialog.DialogCode.Accepted:
        if login.role_id == 1:  # Администратор
            window = AdminWindow(login.username)
            window.show()
            sys.exit(app.exec())
        elif login.role_id == 2:  # Менеджер
            window = ManagerWindow(login.username)
            window.show()
            sys.exit(app.exec())
        elif login.role_id == 3 and login.username == "Гость":  # Гость
            window = GuestWindow(login.username)
            window.show()
            sys.exit(app.exec())
        else:  # Обычный клиент
            window = ClientWindow(login.username)
            window.show()
            sys.exit(app.exec())
