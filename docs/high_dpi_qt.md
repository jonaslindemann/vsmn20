# Fixing Qt resoultion on high DPI displays

Sometime the Qt application does not scale properly on high DPI displays and displays with a very small window and font size. This can be fixed by setting the following attributes before creating the QApplication instance.


```python
# -*- coding: utf-8 -*-

import sys

from qtpy.QtCore import QThread, Qt
from qtpy.QtWidgets import QApplication, QMainWindow, QFileDialog
from qtpy import uic

if __name__ == '__main__':

    """Application main program"""

    # Instantiate application object

    QApplication.setAttribute(Qt.AA_EnableHighDpiScaling, True)
    QApplication.setAttribute(Qt.AA_UseHighDpiPixmaps, True)
    
    app = QApplication(sys.argv)

    # Create window

    window = MyWindow()
    window.show()

    # Application event loop

    sys.exit(app.exec_())
```

