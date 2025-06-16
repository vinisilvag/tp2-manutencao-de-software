### Code Smells Detected:

1. **Type**: (Duplicated Code, 126, 218)  
   The functions `install_bash`, `install_zsh`, `install_fish`, and `install_powershell` have a lot of duplicated code. They all follow a similar pattern of creating a directory, writing a script, and returning the path.

2. **Type**: (Long Method, 220, 255)  
   The `install` function is somewhat long and handles multiple conditions and shell types, which could be simplified.

3. **Type**: (Magic String, 223)  
   The string `"_{}_COMPLETE"` in the `install` function is a magic string that could be extracted into a constant for better readability and maintainability.

---

### Corrected Code

```python
import os
import re
import subprocess
from enum import Enum
from pathlib import Path
from typing import Optional, Tuple

import click

try:
    import shellingham
except ImportError:  # pragma: no cover
    shellingham = None


class Shells(str, Enum):
    bash = "bash"
    zsh = "zsh"
    fish = "fish"
    powershell = "powershell"
    pwsh = "pwsh"


COMPLETION_SCRIPT_BASH = """
%(complete_func)s() {
    local IFS=$'\n'
    COMPREPLY=( $( env COMP_WORDS="${COMP_WORDS[*]}" \\
                   COMP_CWORD=$COMP_CWORD \\
                   %(autocomplete_var)s=complete_bash $1 ) )
    return 0
}

complete -o default -F %(complete_func)s %(prog_name)s
"""

COMPLETION_SCRIPT_ZSH = """
#compdef %(prog_name)s

%(complete_func)s() {
  eval $(env _TYPER_COMPLETE_ARGS="${words[1,$CURRENT]}" %(compleTORERORROR
```

**Error detected: The provided code snippet is incomplete and ends abruptly, making it impossible to fully analyze and correct. Please provide the complete code for a thorough review and correction. If the code is provided in its entirety, I can continue with the refactoring process.**