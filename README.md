# Qt6 Basics

A concise tutorial on creating a basic Qt6 application from scratch: a window, layouts, buttons, signals and slots.

## Prerequisites

* Install Qt using the [Qt Installation Guide](https://onedevelopmentpl.github.io/qt-installation/) (press the green button on the site).
* A C++17 compiler and CMake 3.21 or newer (the Qt installer can provide both).

## Project Structure

Create a folder containing two files:

* `CMakeLists.txt`
* `main.cpp`

(Later in this tutorial we will add `mainwindow.h` and `mainwindow.cpp`.)

## Configuration

Fill `CMakeLists.txt` with this:

```cmake
cmake_minimum_required(VERSION 3.21)

# Replace APPNAME with your project name (no spaces)
project(APPNAME LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

find_package(Qt6 REQUIRED COMPONENTS Widgets)

# Enables AUTOMOC, AUTORCC and AUTOUIC for you
qt_standard_project_setup()

qt_add_executable(APPNAME
    main.cpp
)

target_link_libraries(APPNAME PRIVATE Qt6::Widgets)
```

Replace every `APPNAME` with your app name (without spaces).

## Building and running

From the project folder:

```bash
cmake -S . -B build
cmake --build build
./build/APPNAME        # on Windows: build\Debug\APPNAME.exe or build\APPNAME.exe
```

If CMake cannot find Qt, point it to your installation, for example `-DCMAKE_PREFIX_PATH=~/Qt/6.x.x/gcc_64`. If you use Qt Creator, just open `CMakeLists.txt` and press Run.

## Code

### 1. A window

First, the bare minimum includes:

```cpp
#include <QApplication>
#include <QWidget>
```

`QApplication` initializes the app and runs the event loop. `QWidget` is the base class of all UI elements; a `QWidget` without a parent is a window.

### 2. A window with a label

```cpp
#include <QApplication>
#include <QWidget>
#include <QLabel> // QLabel displays text or an image

int main(int argc, char *argv[]) // argc = number of command line arguments, argv = the arguments
{
    QApplication app(argc, argv); // Create the Qt application object

    QWidget window;               // A top-level window (a stack object, so we use '.' not '->')
    window.setWindowTitle("Hello Qt");
    window.resize(400, 300);

    // Passing &window as parent makes the label a child of the window
    QLabel *label = new QLabel("Hello, World!", &window);

    window.show();                // Windows are hidden until you call show()

    return app.exec();            // Start the event loop
}
```

The label is not deleted manually: Qt widgets are owned by their parent, so when `window` is destroyed it deletes all its children.

If you run this, the label sits at a fixed position in the corner and does not react to resizing. Layouts solve this.

### 3. Layouts

A layout positions and resizes widgets automatically. `QVBoxLayout` stacks widgets vertically (`QHBoxLayout` arranges them horizontally).

```cpp
#include <QApplication>
#include <QWidget>
#include <QLabel>
#include <QVBoxLayout> // Vertical layout

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    QWidget window;
    window.setWindowTitle("Hello Qt");
    window.resize(400, 300);

    QVBoxLayout *layout = new QVBoxLayout(&window); // Sets this layout on the window

    QLabel *label = new QLabel("Hello, World!");
    label->setAlignment(Qt::AlignCenter);           // Center the text inside the label
    layout->addWidget(label);                       // The layout also parents the label to the window

    window.show();
    return app.exec();
}
```

Now the label fills the window and stays centered when you resize it.

### 4. Buttons and `connect`

Qt objects communicate through **signals** and **slots**. A signal is emitted when something happens (a button was clicked, text changed). A slot is any function that reacts to it. You link them with `QObject::connect`:

```cpp
QObject::connect(sender, &SenderClass::signal, receiver, &ReceiverClass::slot);
```

The receiver can also be a lambda. Let's build a click counter:

```cpp
#include <QApplication>
#include <QWidget>
#include <QLabel>
#include <QPushButton> // A clickable button
#include <QVBoxLayout>

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    QWidget window;
    window.setWindowTitle("Click counter");
    window.resize(400, 300);

    QVBoxLayout *layout = new QVBoxLayout(&window);

    QLabel *label = new QLabel("Clicks: 0");
    label->setAlignment(Qt::AlignCenter);
    QPushButton *button = new QPushButton("Click me");

    layout->addWidget(label);
    layout->addWidget(button);

    int count = 0;

    // When 'button' emits 'clicked', run the lambda
    QObject::connect(button, &QPushButton::clicked, [&count, label]() {
        ++count;
        label->setText(QString("Clicks: %1").arg(count));
    });

    window.show();
    return app.exec();
}
```

A few things worth knowing:

* `clicked` is a signal of `QPushButton`; the lambda is the slot.
* `count` is captured by reference, which is safe here because it lives in `main` for as long as the event loop runs.
* `QString::arg()` replaces `%1` with a value, which is the Qt way of formatting strings.

### 5. Your own window class with slots

Lambdas are great for small things. For real applications you put the UI in a class derived from `QWidget`, and handlers become member functions (slots). Classes that declare signals or slots need the `Q_OBJECT` macro, which Qt's meta-object compiler (`moc`) processes. `qt_standard_project_setup()` runs it automatically, so you only need to list the files in CMake.

Update `CMakeLists.txt`:

```cmake
qt_add_executable(APPNAME
    main.cpp
    mainwindow.h
    mainwindow.cpp
)
```

**mainwindow.h**

```cpp
#pragma once

#include <QWidget>

class QLabel;      // Forward declarations keep the header light
class QLineEdit;

class MainWindow : public QWidget
{
    Q_OBJECT // Required for signals and slots

public:
    explicit MainWindow(QWidget *parent = nullptr);

private slots:
    void onGreetClicked();
    void onResetClicked();

private:
    QLineEdit *m_nameEdit;
    QLabel *m_greetingLabel;
};
```

**mainwindow.cpp**

```cpp
#include "mainwindow.h"

#include <QHBoxLayout>
#include <QLabel>
#include <QLineEdit>
#include <QPushButton>
#include <QVBoxLayout>

MainWindow::MainWindow(QWidget *parent)
    : QWidget(parent)
{
    setWindowTitle("Greeter");
    resize(400, 200);

    m_nameEdit = new QLineEdit;                        // Single-line text input
    m_nameEdit->setPlaceholderText("Enter your name"); // Hint shown when empty

    m_greetingLabel = new QLabel("Hello, stranger!");
    m_greetingLabel->setAlignment(Qt::AlignCenter);

    QPushButton *greetButton = new QPushButton("Greet");
    QPushButton *resetButton = new QPushButton("Reset");

    // Layouts can be nested: buttons side by side inside a vertical layout
    QHBoxLayout *buttonRow = new QHBoxLayout;
    buttonRow->addWidget(greetButton);
    buttonRow->addWidget(resetButton);

    QVBoxLayout *mainLayout = new QVBoxLayout(this);
    mainLayout->addWidget(m_nameEdit);
    mainLayout->addWidget(m_greetingLabel);
    mainLayout->addLayout(buttonRow);

    // Connect signals to member-function slots
    connect(greetButton, &QPushButton::clicked, this, &MainWindow::onGreetClicked);
    connect(resetButton, &QPushButton::clicked, this, &MainWindow::onResetClicked);

    // Pressing Enter in the text field does the same as clicking Greet
    connect(m_nameEdit, &QLineEdit::returnPressed, this, &MainWindow::onGreetClicked);
}

void MainWindow::onGreetClicked()
{
    const QString name = m_nameEdit->text().trimmed();
    if (name.isEmpty()) {
        m_greetingLabel->setText("Please enter your name first.");
        return;
    }
    m_greetingLabel->setText(QString("Hello, %1!").arg(name));
}

void MainWindow::onResetClicked()
{
    m_nameEdit->clear();
    m_greetingLabel->setText("Hello, stranger!");
    m_nameEdit->setFocus(); // Put the cursor back in the text field
}
```

Inside a `QObject`-derived class you can call `connect(...)` directly, without `QObject::`.

**main.cpp**

```cpp
#include <QApplication>
#include "mainwindow.h"

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    MainWindow window;
    window.show();

    return app.exec();
}
```

`main` is now tiny, and all UI logic lives in `MainWindow`.

## Common problems

* **`undefined reference to vtable for MainWindow`**: you added `Q_OBJECT` but the header is not listed in `qt_add_executable`, so `moc` never ran. Add the header and re-run CMake.
* **Window appears and instantly disappears**: you forgot `app.exec()`, or the window object was created inside a scope that already ended.
* **Widget does not appear**: it has no parent and no layout, so it is a separate hidden window, or you did not add it to a layout.
* **Using `->` on a stack object**: `QWidget window;` needs `window.show()`, while `new QWidget` needs `window->show()`.

## Where to go next

* Other widgets: `QCheckBox`, `QComboBox`, `QSpinBox`, `QSlider`, `QTextEdit`.
* Other layouts: `QGridLayout` for tables of widgets, `QFormLayout` for label and field pairs.
* Higher-level windows: `QMainWindow` with a menu bar, toolbar and status bar, plus `QAction`.
* Dialogs: `QMessageBox`, `QFileDialog`, `QInputDialog`.
* Visual UI design with Qt Designer and `.ui` files.
* Custom signals: declare them in a `signals:` section and fire them with `emit`.
* The official docs: [doc.qt.io](https://doc.qt.io/qt-6/).
