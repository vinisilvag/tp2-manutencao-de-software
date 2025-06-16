Code Smells Detected:
1. **Type**: (Duplicated Code, 15, 17)
2. **Type**: (Poor Naming, 15, 17)

The code contains repetitive functions (`get_red`, `get_green`, `get_blue`) that essentially return a constant value. Additionally, the function names are not particularly meaningful since they simply return a constant.

Corrected Code
====== [CORRECTED CODE START] =======
# Variable names to colors, just for completion
BLACK = "black"
RED = "red"
GREEN = "green"
YELLOW = "yellow"
BLUE = "blue"
MAGENTA = "magenta"
CYAN = "cyan"
WHITE = "white"

RESET = "reset"

BRIGHT_BLACK = "bright_black"
BRIGHT_RED = "bright_red"
BRIGHT_GREEN = "bright_green"
BRIGHT_YELLOW = "bright_yellow"
BRIGHT_BLUE = "bright_blue"
BRIGHT_MAGENTA = "bright_magenta"
BRIGHT_CYAN = "bright_cyan"
BRIGHT_WHITE = "bright_white"

====== [ADDED CODE] =======
def get_color(color_name):
    """
    Returns the color value based on the provided color name.
    :param color_name: Name of the color (e.g., 'red', 'green', 'blue')
    :return: Corresponding color value
    """
    return globals().get(color_name.upper(), None)
====== [CORRECTED CODE END] =======

### Explanation:
- **Duplicated Code**: The functions `get_red`, `get_green`, and `get_blue` were removed because they were simply returning a constant value, which is redundant.
- **Poor Naming**: The function names (`get_red`, `get_green`, `get_blue`) were replaced with a more generic and meaningful function `get_color`, which can handle any color name dynamically using `globals()`. This makes the code cleaner and more maintainable.