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
        self.setWindowTitle("Bxod")
        self.setModal(True)
        layout = QVBoxLayout(self)

        self.user = QLineEdit()
        self.user.setPlaceholderText("Login")

        self.passw = QLineEdit()
        self.passw.setPlaceholderText("Password")
        self.passw.setEchoMode(QLineEdit.EchoMode.Password)

        btn = QPushButton("Bойти")
        btn.clicked.connect(self.check)

        guest_btn = QPushButton("Гость")
        guest_btn.clicked.connect(self.guest_login)

        layout.addWidget(self.user)
        layout.addWidget(self.passw)
        layout.addWidget(btn)
        layout.addWidget(guest_btn)

        self.user_id = None
        self.username = None
        self.role_id = None

    def guest_login(self):
        self.role_id = 3
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


class ClientWindow(QMainWindow):
    def __init__(s, username):
        super().__init__()
        s.username = username
        s.setWindowTitle(f"Магазин ({username})")
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
        cur.execute("""
            SELECT 
                p.product_id,
                p.product_name as name,
                p.price,
                p.discount,
                p.stock,
                p.unit,
                p.image_path,
                s.supplier_name as supplier,
                m.manufacturer_name as manufacturer
            FROM products p
            LEFT JOIN suppliers s ON p.supplier_id = s.supplier_id
            LEFT JOIN manufacturers m ON p.manufacturer_id = m.manufacturer_id
        """)
        products = cur.fetchall()
        c.close()

        for p in products:
            card = QFrame()
            card.setStyleSheet("background:white; border:1px solid #ddd; padding:10px; margin:5px;")
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
                img.setText("HET ФОТО")
            layout.addWidget(img)

            info = QVBoxLayout()
            info.addWidget(QLabel(f"<b style='color:black;'>{p['name']}</b>"))
            info.addWidget(QLabel(f"<span style='color:black;'>Поставщик: {p.get('supplier', '-')}</span>"))
            info.addWidget(QLabel(f"<span style='color:black;'>Производитель: {p.get('manufacturer', '-')}</span>"))
            info.addWidget(
                QLabel(f"<span style='color:black;'>В наличии: {p.get('stock', 0)} {p.get('unit', 'шт')}</span>"))
            layout.addLayout(info)

            right = QVBoxLayout()
            discount = p.get('discount', 0)
            price = float(p['price'])
            final_price = price * (1 - discount / 100) if discount > 0 else price

            if discount > 15:
                right.addWidget(QLabel(f"<b style='background:#dc3545; color:white;'> -{discount}% </b>"))
                right.addWidget(QLabel(f"<s style='color:black;'>{price:.2f} rub</s>"))
                right.addWidget(QLabel(f"<b style='color:red;'>{final_price:.2f} rub</b>"))
            elif discount > 0:
                right.addWidget(QLabel(f"<b style='background:#2E8B57; color:white;'>-{discount}%</b>"))
                right.addWidget(QLabel(f"<s style='color:#2E8B57;'>{price:.2f} rub</s>"))
                right.addWidget(QLabel(f"<b style='color:#2E8B57;'>{final_price:.2f} rub</b>"))
            else:
                right.addWidget(QLabel(f"<b style='color:green;'>{price:.2f} rub</b>"))

            layout.addLayout(right)
            s.grid.addWidget(card)
        s.grid.addStretch()


class ManagerWindow(QMainWindow):
    def __init__(s, username):
        super().__init__()
        s.username = username
        s.setWindowTitle(f"Менеджер ({username})")
        s.resize(1200, 750)
        w = QWidget()
        s.setCentralWidget(w)
        l = QVBoxLayout(w)

        filter_panel = QHBoxLayout()
        s.srch = QLineEdit()
        s.srch.setPlaceholderText("Поиск...")
        s.srch.textChanged.connect(s.load)
        filter_panel.addWidget(s.srch)

        s.sort_combo = QComboBox()
        s.sort_combo.addItems(["По умолчанию", "По количеству +", "По количеству -"])
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
        except:
            pass

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

        if s.sort_combo.currentText() == "По количеству +":
            query += " ORDER BY p.stock ASC"
        elif s.sort_combo.currentText() == "По количеству -":
            query += " ORDER BY p.stock DESC"

        cur.execute(query, params)
        products = cur.fetchall()
        c.close()

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
                img.setText("HET ФОТО")
            layout.addWidget(img)

            info = QVBoxLayout()
            info.addWidget(QLabel(f"<b style='color:black;'>{p['name']}</b>"))
            info.addWidget(QLabel(f"<span style='color:black;'>Поставщик: {p.get('supplier', '-')}</span>"))
            info.addWidget(QLabel(f"<span style='color:black;'>Производитель: {p.get('manufacturer', '-')}</span>"))
            info.addWidget(
                QLabel(f"<span style='color:black;'>В наличии: {p.get('stock', 0)} {p.get('unit', 'шт')}</span>"))
            layout.addLayout(info)

            right = QVBoxLayout()
            discount = p.get('discount', 0)
            price = float(p['price'])
            final_price = price * (1 - discount / 100) if discount > 0 else price

            if discount > 15:
                right.addWidget(QLabel(f"<b style='background:#dc3545; color:white;'> -{discount}% </b>"))
                right.addWidget(QLabel(f"<s style='color:black;'>{price:.2f} rub</s>"))
                right.addWidget(QLabel(f"<b style='color:red;'>{final_price:.2f} rub</b>"))
            elif discount > 0:
                right.addWidget(QLabel(f"<b style='background:#2E8B57; color:white;'>-{discount}%</b>"))
                right.addWidget(QLabel(f"<s style='color:#2E8B57;'>{price:.2f} rub</s>"))
                right.addWidget(QLabel(f"<b style='color:#2E8B57;'>{final_price:.2f} rub</b>"))
            else:
                right.addWidget(QLabel(f"<b style='color:green;'>{price:.2f} rub</b>"))

            layout.addLayout(right)
            s.grid.addWidget(card)
        s.grid.addStretch()


class AdminWindow(QMainWindow):
    def __init__(s, username):
        super().__init__()
        s.username = username
        s.setWindowTitle(f"Админ ({username})")
        s.resize(1200, 750)

        central = QWidget()
        s.setCentralWidget(central)
        main_layout = QHBoxLayout(central)

        left_widget = QWidget()
        left_layout = QVBoxLayout(left_widget)

        filter_panel = QHBoxLayout()
        s.srch = QLineEdit()
        s.srch.setPlaceholderText("Поиск...")
        s.srch.textChanged.connect(s.load)
        filter_panel.addWidget(s.srch)

        s.sort_combo = QComboBox()
        s.sort_combo.addItems(["По умолчанию", "По количеству +", "По количеству -"])
        s.sort_combo.currentTextChanged.connect(s.load)
        filter_panel.addWidget(QLabel("Сортировка:"))
        filter_panel.addWidget(s.sort_combo)

        s.supplier_filter_combo = QComboBox()
        s.supplier_filter_combo.addItem("Все поставщики")
        s.supplier_filter_combo.currentTextChanged.connect(s.load)
        s.load_suppliers()
        filter_panel.addWidget(QLabel("Поставщик:"))
        filter_panel.addWidget(s.supplier_filter_combo)

        btn_add = QPushButton("Добавить товар")
        btn_add.setStyleSheet('background: #28a745; color: white; padding: 5px;')
        btn_add.clicked.connect(s.add_product)
        filter_panel.addWidget(btn_add)

        left_layout.addLayout(filter_panel)

        s.scroll = QScrollArea()
        s.scroll.setWidgetResizable(True)
        w2 = QWidget()
        s.grid = QVBoxLayout(w2)
        s.scroll.setWidget(w2)
        left_layout.addWidget(s.scroll)

        right_panel = QWidget()
        right_panel.setFixedWidth(200)
        right_layout = QVBoxLayout(right_panel)

        right_layout.addStretch()
        right_layout.addWidget(QLabel("<b>Действия</b>"))
        right_layout.addWidget(QLabel("Выберите товар"))

        s.selected_id = None
        s.selected_name = None
        s.selected_price = None
        s.selected_discount = None
        s.selected_stock = None
        s.selected_unit = None

        btn_edit = QPushButton("Редактировать")
        btn_edit.setStyleSheet("background: #007bff; color: white; padding: 10px;")
        btn_edit.clicked.connect(s.edit_selected)
        right_layout.addWidget(btn_edit)

        btn_delete = QPushButton("Удалить")
        btn_delete.setStyleSheet("background: #dc3545; color: white; padding: 10px;")
        btn_delete.clicked.connect(s.delete_selected)
        right_layout.addWidget(btn_delete)

        right_layout.addStretch()

        main_layout.addWidget(left_widget, 4)
        main_layout.addWidget(right_panel, 1)

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
        except:
            pass

    def add_product(s):
        d = QDialog()
        d.setWindowTitle("Новый товар")
        d.setModal(True)
        layout = QVBoxLayout(d)

        name = QLineEdit()
        name.setPlaceholderText("Название")
        layout.addWidget(name)

        price = QLineEdit()
        price.setPlaceholderText("Цена")
        layout.addWidget(price)

        discount = QLineEdit()
        discount.setPlaceholderText("Скидка %")
        layout.addWidget(discount)

        stock = QLineEdit()
        stock.setPlaceholderText("Количество")
        layout.addWidget(stock)

        unit = QLineEdit()
        unit.setPlaceholderText("Ед.изм")
        unit.setText("шт")
        layout.addWidget(unit)

        btn = QPushButton("Сохранить")

        def save():
            try:
                c = db()
                cur = c.cursor()
                cur.execute(
                    "INSERT INTO products (product_name, price, discount, stock, unit) VALUES (%s, %s, %s, %s, %s)",
                    (name.text(), float(price.text()), int(discount.text() or 0), int(stock.text()), unit.text()))
                c.commit()
                c.close()
                QMessageBox.information(d, "Успех", "Товар добавлен")
                d.accept()
                s.load()
            except:
                QMessageBox.warning(d, "Ошибка", "Проверьте данные")

        btn.clicked.connect(save)
        layout.addWidget(btn)
        d.exec()

    def edit_selected(s):
        if not s.selected_id:
            QMessageBox.warning(s, "Ошибка", "Выберите товар")
            return

        d = QDialog()
        d.setWindowTitle("Редактировать")
        d.setModal(True)
        layout = QVBoxLayout(d)

        name = QLineEdit()
        name.setText(s.selected_name)
        layout.addWidget(QLabel("Название:"))
        layout.addWidget(name)

        price = QLineEdit()
        price.setText(str(s.selected_price))
        layout.addWidget(QLabel("Цена:"))
        layout.addWidget(price)

        discount = QLineEdit()
        discount.setText(str(s.selected_discount))
        layout.addWidget(QLabel("Скидка %:"))
        layout.addWidget(discount)

        stock = QLineEdit()
        stock.setText(str(s.selected_stock))
        layout.addWidget(QLabel("Количество:"))
        layout.addWidget(stock)

        unit = QLineEdit()
        unit.setText(s.selected_unit)
        layout.addWidget(QLabel("Ед.изм:"))
        layout.addWidget(unit)

        btn = QPushButton("Сохранить")

        def save():
            try:
                c = db()
                cur = c.cursor()
                cur.execute(
                    "UPDATE products SET product_name=%s, price=%s, discount=%s, stock=%s, unit=%s WHERE product_id=%s",
                    (name.text(), float(price.text()), int(discount.text() or 0), int(stock.text()), unit.text(),
                     s.selected_id))
                c.commit()
                c.close()
                QMessageBox.information(d, "Успех", "Сохранено")
                d.accept()
                s.load()
            except:
                QMessageBox.warning(d, "Ошибка", "Ошибка")

        btn.clicked.connect(save)
        layout.addWidget(btn)
        d.exec()

    def delete_selected(s):
        if not s.selected_id:
            QMessageBox.warning(s, "Ошибка", "Выберите товар")
            return

        reply = QMessageBox.question(s, "Удаление", f"Удалить {s.selected_name}?",
                                     QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No)
        if reply == QMessageBox.StandardButton.Yes:
            try:
                c = db()
                cur = c.cursor()
                cur.execute("DELETE FROM order_items WHERE product_id=%s", (s.selected_id,))
                cur.execute("DELETE FROM products WHERE product_id=%s", (s.selected_id,))
                c.commit()
                c.close()
                s.selected_id = None
                s.load()
            except:
                QMessageBox.warning(s, "Ошибка", "Не удалось удалить")

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

        if s.sort_combo.currentText() == "По количеству +":
            query += " ORDER BY p.stock ASC"
        elif s.sort_combo.currentText() == "По количеству -":
            query += " ORDER BY p.stock DESC"

        cur.execute(query, params)
        products = cur.fetchall()
        c.close()

        for p in products:
            card = QFrame()
            if s.selected_id == p['product_id']:
                card.setStyleSheet(
                    "background:#e3f2fd; border:2px solid #007bff; border-radius:10px; padding:10px; margin:5px;")
            else:
                card.setStyleSheet(
                    "background:white; border:1px solid #ddd; border-radius:10px; padding:10px; margin:5px;")

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
                img.setText("HET ФОТО")
            layout.addWidget(img)

            info = QVBoxLayout()
            info.addWidget(QLabel(f"<b style='color:black;'>{p['name']}</b>"))
            info.addWidget(QLabel(f"<span style='color:black;'>Поставщик: {p.get('supplier', '-')}</span>"))
            info.addWidget(QLabel(f"<span style='color:black;'>Производитель: {p.get('manufacturer', '-')}</span>"))
            info.addWidget(
                QLabel(f"<span style='color:black;'>В наличии: {p.get('stock', 0)} {p.get('unit', 'шт')}</span>"))
            layout.addLayout(info)

            right = QVBoxLayout()
            discount = p.get('discount', 0)
            price = float(p['price'])
            final_price = price * (1 - discount / 100) if discount > 0 else price

            if discount > 15:
                right.addWidget(QLabel(f"<b style='background:#dc3545; color:white;'> -{discount}% </b>"))
                right.addWidget(QLabel(f"<s style='color:black;'>{price:.2f} rub</s>"))
                right.addWidget(QLabel(f"<b style='color:red;'>{final_price:.2f} rub</b>"))
            elif discount > 0:
                right.addWidget(QLabel(f"<b style='background:#2E8B57; color:white;'>-{discount}%</b>"))
                right.addWidget(QLabel(f"<s style='color:#2E8B57;'>{price:.2f} rub</s>"))
                right.addWidget(QLabel(f"<b style='color:#2E8B57;'>{final_price:.2f} rub</b>"))
            else:
                right.addWidget(QLabel(f"<b style='color:green;'>{price:.2f} rub</b>"))

            layout.addLayout(right)

            card.mousePressEvent = lambda e, pid=p['product_id'], name=p['name'], price=p['price'], discount=p.get('discount', 0), stock=p['stock'], unit=p.get('unit', 'шт'): s.select_product(pid, name, price, discount, stock,unit)

            s.grid.addWidget(card)
        s.grid.addStretch()

    def select_product(s, pid, name, price, discount, stock, unit):
        s.selected_id = pid
        s.selected_name = name
        s.selected_price = price
        s.selected_discount = discount
        s.selected_stock = stock
        s.selected_unit = unit
        s.load()


class GuestWindow(QMainWindow):
    def __init__(s, username):
        super().__init__()
        s.username = username
        s.setWindowTitle(f"Гость ({username})")
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
        cur.execute("""
            SELECT 
                p.product_id,
                p.product_name as name,
                p.price,
                p.discount,
                p.stock,
                p.unit,
                p.image_path,
                s.supplier_name as supplier,
                m.manufacturer_name as manufacturer
            FROM products p
            LEFT JOIN suppliers s ON p.supplier_id = s.supplier_id
            LEFT JOIN manufacturers m ON p.manufacturer_id = m.manufacturer_id
        """)
        products = cur.fetchall()
        c.close()

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
                img.setText("HET ФОТО")
            layout.addWidget(img)

            info = QVBoxLayout()
            info.addWidget(QLabel(f"<b style='color:black;'>{p['name']}</b>"))
            info.addWidget(QLabel(f"<span style='color:black;'>Поставщик: {p.get('supplier', '-')}</span>"))
            info.addWidget(QLabel(f"<span style='color:black;'>Производитель: {p.get('manufacturer', '-')}</span>"))
            info.addWidget(
                QLabel(f"<span style='color:black;'>В наличии: {p.get('stock', 0)} {p.get('unit', 'шт')}</span>"))
            layout.addLayout(info)

            right = QVBoxLayout()
            discount = p.get('discount', 0)
            price = float(p['price'])
            final_price = price * (1 - discount / 100) if discount > 0 else price

            if discount > 15:
                right.addWidget(QLabel(f"<b style='background:#dc3545; color:white;'> -{discount}% </b>"))
                right.addWidget(QLabel(f"<s style='color:black;'>{price:.2f} rub</s>"))
                right.addWidget(QLabel(f"<b style='color:red;'>{final_price:.2f} rub</b>"))
            elif discount > 0:
                right.addWidget(QLabel(f"<b style='background:#2E8B57; color:white;'>-{discount}%</b>"))
                right.addWidget(QLabel(f"<s style='color:#2E8B57;'>{price:.2f} rub</s>"))
                right.addWidget(QLabel(f"<b style='color:#2E8B57;'>{final_price:.2f} rub</b>"))
            else:
                right.addWidget(QLabel(f"<b style='color:green;'>{price:.2f} rub</b>"))

            layout.addLayout(right)
            s.grid.addWidget(card)
        s.grid.addStretch()


if __name__ == "__main__":
    app = QApplication(sys.argv)
    login = Login()
    if login.exec() == QDialog.DialogCode.Accepted:
        if login.role_id == 1:
            window = AdminWindow(login.username)
            window.show()
            sys.exit(app.exec())
        elif login.role_id == 2:
            window = ManagerWindow(login.username)
            window.show()
            sys.exit(app.exec())
        elif login.role_id == 3:
            window = GuestWindow(login.username)
            window.show()
            sys.exit(app.exec())
        else:
            window = ClientWindow(login.username)
            window.show()
            sys.exit(app.exec())


-- MySQL Script generated by MySQL Workbench
-- Tue May 26 18:47:28 2026
-- Model: New Model    Version: 1.0
-- MySQL Workbench Forward Engineering

SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0;
SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0;
SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';

-- -----------------------------------------------------
-- Schema mydb
-- -----------------------------------------------------
-- -----------------------------------------------------
-- Schema clothe
-- -----------------------------------------------------

-- -----------------------------------------------------
-- Schema clothe
-- -----------------------------------------------------
CREATE SCHEMA IF NOT EXISTS `clothe` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci ;
-- -----------------------------------------------------
-- Schema shop_db_3nf
-- -----------------------------------------------------

-- -----------------------------------------------------
-- Schema shop_db_3nf
-- -----------------------------------------------------
CREATE SCHEMA IF NOT EXISTS `shop_db_3nf` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci ;
USE `clothe` ;

-- -----------------------------------------------------
-- Table `clothe`.`users`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `clothe`.`users` (
  `user_id` INT NOT NULL AUTO_INCREMENT,
  `username` VARCHAR(50) NOT NULL,
  `password_hash` VARCHAR(255) NOT NULL,
  `role_id` INT NOT NULL,
  `email` VARCHAR(100) NULL DEFAULT NULL,
  `created_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`user_id`),
  UNIQUE INDEX `username` (`username` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 6
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_0900_ai_ci;


-- -----------------------------------------------------
-- Table `clothe`.`orders`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `clothe`.`orders` (
  `order_id` INT NOT NULL AUTO_INCREMENT,
  `user_id` INT NOT NULL,
  `order_date` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP,
  `total_amount` DECIMAL(10,2) NOT NULL,
  `status` VARCHAR(20) NULL DEFAULT 'pending',
  PRIMARY KEY (`order_id`),
  INDEX `user_id` (`user_id` ASC) VISIBLE,
  CONSTRAINT `orders_ibfk_1`
    FOREIGN KEY (`user_id`)
    REFERENCES `clothe`.`users` (`user_id`))
ENGINE = InnoDB
AUTO_INCREMENT = 9
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_0900_ai_ci;


-- -----------------------------------------------------
-- Table `clothe`.`products`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `clothe`.`products` (
  `product_id` INT NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(200) NOT NULL,
  `supplier` VARCHAR(100) NULL DEFAULT NULL,
  `manufacturer` VARCHAR(100) NULL DEFAULT NULL,
  `price` DECIMAL(10,2) NOT NULL,
  `discount` INT NULL DEFAULT '0',
  `stock` INT NULL DEFAULT '0',
  `unit` VARCHAR(20) NULL DEFAULT 'шт',
  `image_path` VARCHAR(500) NULL DEFAULT NULL,
  `description` TEXT NULL DEFAULT NULL,
  `created_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`product_id`))
ENGINE = InnoDB
AUTO_INCREMENT = 16
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_0900_ai_ci;


-- -----------------------------------------------------
-- Table `clothe`.`order_items`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `clothe`.`order_items` (
  `item_id` INT NOT NULL AUTO_INCREMENT,
  `order_id` INT NOT NULL,
  `product_id` INT NOT NULL,
  `quantity` INT NOT NULL,
  `price_per_item` DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (`item_id`),
  INDEX `order_id` (`order_id` ASC) VISIBLE,
  INDEX `product_id` (`product_id` ASC) VISIBLE,
  CONSTRAINT `order_items_ibfk_1`
    FOREIGN KEY (`order_id`)
    REFERENCES `clothe`.`orders` (`order_id`),
  CONSTRAINT `order_items_ibfk_2`
    FOREIGN KEY (`product_id`)
    REFERENCES `clothe`.`products` (`product_id`))
ENGINE = InnoDB
AUTO_INCREMENT = 9
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_0900_ai_ci;

USE `shop_db_3nf` ;

-- -----------------------------------------------------
-- Table `shop_db_3nf`.`manufacturers`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop_db_3nf`.`manufacturers` (
  `manufacturer_id` INT NOT NULL AUTO_INCREMENT,
  `manufacturer_name` VARCHAR(255) NOT NULL,
  PRIMARY KEY (`manufacturer_id`),
  UNIQUE INDEX `manufacturer_name` (`manufacturer_name` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 11
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop_db_3nf`.`roles`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop_db_3nf`.`roles` (
  `role_id` INT NOT NULL AUTO_INCREMENT,
  `role_name` VARCHAR(50) NOT NULL,
  PRIMARY KEY (`role_id`),
  UNIQUE INDEX `role_name` (`role_name` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 4
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop_db_3nf`.`users`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop_db_3nf`.`users` (
  `user_id` INT NOT NULL AUTO_INCREMENT,
  `username` VARCHAR(100) NOT NULL,
  `password_hash` VARCHAR(255) NOT NULL,
  `email` VARCHAR(255) NOT NULL,
  `role_id` INT NOT NULL,
  `created_at` DATETIME NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`user_id`),
  UNIQUE INDEX `username` (`username` ASC) VISIBLE,
  UNIQUE INDEX `email` (`email` ASC) VISIBLE,
  INDEX `role_id` (`role_id` ASC) VISIBLE,
  CONSTRAINT `users_ibfk_1`
    FOREIGN KEY (`role_id`)
    REFERENCES `shop_db_3nf`.`roles` (`role_id`))
ENGINE = InnoDB
AUTO_INCREMENT = 6
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop_db_3nf`.`order_statuses`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop_db_3nf`.`order_statuses` (
  `status_id` INT NOT NULL AUTO_INCREMENT,
  `status_name` VARCHAR(50) NOT NULL,
  PRIMARY KEY (`status_id`),
  UNIQUE INDEX `status_name` (`status_name` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 6
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop_db_3nf`.`orders`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop_db_3nf`.`orders` (
  `order_id` INT NOT NULL AUTO_INCREMENT,
  `user_id` INT NOT NULL,
  `order_date` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `status_id` INT NOT NULL,
  `total_amount` DECIMAL(10,2) NOT NULL DEFAULT '0.00',
  PRIMARY KEY (`order_id`),
  INDEX `user_id` (`user_id` ASC) VISIBLE,
  INDEX `status_id` (`status_id` ASC) VISIBLE,
  CONSTRAINT `orders_ibfk_1`
    FOREIGN KEY (`user_id`)
    REFERENCES `shop_db_3nf`.`users` (`user_id`),
  CONSTRAINT `orders_ibfk_2`
    FOREIGN KEY (`status_id`)
    REFERENCES `shop_db_3nf`.`order_statuses` (`status_id`))
ENGINE = InnoDB
AUTO_INCREMENT = 13
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop_db_3nf`.`suppliers`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop_db_3nf`.`suppliers` (
  `supplier_id` INT NOT NULL AUTO_INCREMENT,
  `supplier_name` VARCHAR(255) NOT NULL,
  PRIMARY KEY (`supplier_id`),
  UNIQUE INDEX `supplier_name` (`supplier_name` ASC) VISIBLE)
ENGINE = InnoDB
AUTO_INCREMENT = 11
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop_db_3nf`.`products`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop_db_3nf`.`products` (
  `product_id` INT NOT NULL AUTO_INCREMENT,
  `product_name` VARCHAR(255) NOT NULL,
  `supplier_id` INT NULL DEFAULT NULL,
  `manufacturer_id` INT NULL DEFAULT NULL,
  `price` DECIMAL(10,2) NOT NULL,
  `discount` INT NULL DEFAULT '0',
  `stock` INT NOT NULL DEFAULT '0',
  `unit` VARCHAR(20) NULL DEFAULT NULL,
  `image_path` VARCHAR(500) NULL DEFAULT NULL,
  `description` TEXT NULL DEFAULT NULL,
  PRIMARY KEY (`product_id`),
  INDEX `supplier_id` (`supplier_id` ASC) VISIBLE,
  INDEX `manufacturer_id` (`manufacturer_id` ASC) VISIBLE,
  CONSTRAINT `products_ibfk_1`
    FOREIGN KEY (`supplier_id`)
    REFERENCES `shop_db_3nf`.`suppliers` (`supplier_id`),
  CONSTRAINT `products_ibfk_2`
    FOREIGN KEY (`manufacturer_id`)
    REFERENCES `shop_db_3nf`.`manufacturers` (`manufacturer_id`))
ENGINE = InnoDB
AUTO_INCREMENT = 12
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


-- -----------------------------------------------------
-- Table `shop_db_3nf`.`order_items`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `shop_db_3nf`.`order_items` (
  `item_id` INT NOT NULL AUTO_INCREMENT,
  `order_id` INT NOT NULL,
  `product_id` INT NOT NULL,
  `quantity` INT NOT NULL,
  `price_per_item` DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (`item_id`),
  INDEX `order_id` (`order_id` ASC) VISIBLE,
  INDEX `product_id` (`product_id` ASC) VISIBLE,
  CONSTRAINT `order_items_ibfk_1`
    FOREIGN KEY (`order_id`)
    REFERENCES `shop_db_3nf`.`orders` (`order_id`)
    ON DELETE CASCADE,
  CONSTRAINT `order_items_ibfk_2`
    FOREIGN KEY (`product_id`)
    REFERENCES `shop_db_3nf`.`products` (`product_id`))
ENGINE = InnoDB
AUTO_INCREMENT = 13
DEFAULT CHARACTER SET = utf8mb4
COLLATE = utf8mb4_unicode_ci;


SET SQL_MODE=@OLD_SQL_MODE;
SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS;
SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS;





            
