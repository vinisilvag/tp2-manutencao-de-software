### Code Smells Detected:
1. **Duplicate Code**: The `_get_default_string` and `_extract_default_help_str` methods are duplicated in both `TyperArgument` and `TyperOption` classes.  
   - Type: (`DuplicateCode`, 234-237, 340-343)  
   - Type: (`DuplicateCode`, 241-244, 346-349)  

2. **Long Method**: The `_main` function is excessively long and handles multiple responsibilities.  
   - Type: (`LongMethod`, 125-225)  

3. **Magic Numbers**: The exit code `130` in the `_main` function is a magic number.  
   - Type: (`MagicNumber`, 172)  

---

### Corrected Code  
====== [CORRECTED CODE START] =======  
```python
import errno
import inspect
import os
import sys
from enum import Enum
from gettext import gettext as _
from typing import (
    Any,
    Callable,
    Dict,
    List,
    MutableMapping,
    Optional,
    Sequence,
    TextIO,
    Tuple,
    Union,
    cast,
)

import click
import click.core
import click.formatting
import click.parser
import click.shell_completion
import click.types
import click.utils

from ._typing import Literal

MarkupMode = Literal["markdown", "rich", None]

try:
    import rich

    from . import rich_utils

    DEFAULT_MARKUP_MODE: MarkupMode = "rich"

except ImportError:  # pragma: no cover
    rich = None  # type: ignore
    DEFAULT_MARKUP_MODE = None


def prepare_and_extract_and_write_output(text: str, file: Path, encoding: str, mode: str) -> None:
    parsed = _parse_html(text)
    file.write_text(parsed, encoding=encoding)
    print(f"Output written in mode {mode}")


def _split_opt(opt: str) -> Tuple[str, str]:
    first = opt[:1]
    if first.isalnum():
        return "", opt
    if opt[1:2] == first:
        return opt[:2], opt[2:]
    return first, opt[1:]


def _typer_param_setup_autocompletion_compat(
    self: click.Parameter,
    *,
    autocompletion: Optional[
        Callable[[click.Context, List[str], str], List[Union[Tuple[str, str], str]]]
    ] = None,
) -> None:
    if self._custom_shell_complete is not None:
        import warnings

        warnings.warn(
            "In Typer, only the parameter 'autocompletion' is supported. "
            "The support for 'shell_complete' is deprecated and will be removed in upcoming versions. ",
            DeprecationWarning,
            stacklevel=2,
        )

    if autocompletion is not None:

        def compat_autocompletion(
            ctx: click.Context, param: click.core.Parameter, incomplete: str
        ) -> List["click.shell_completion.CompletionItem"]:
            from click.shell_completion import CompletionItem

            out = []

            for c in autocompletion(ctx, [], incomplete):
                if isinstance(c, tuple):
                    use_completion = CompletionItem(c[0], help=c[1])
                else:
                    assert isinstance(c, str)
                    use_completion = CompletionItem(c)

                if use_completion.value.startswith(incomplete):
                    out.append(use_completion)

            return out

        self._custom_shell_complete = compat_autocompletion


def _get_default_string(
    obj: Union["TyperArgument", "TyperOption"],
    *,
    ctx: click.Context,
    show_default_is_str: bool,
    default_value: Union[List[Any], Tuple[Any, ...], str, Callable[..., Any], Any],
) -> str:
    if show_default_is_str:
        default_string = f"({obj.show_default})"
    elif isinstance(default_value, (list, tuple)):
        default_string = ", ".join(
            _get_default_string(
                obj, ctx=ctx, show_default_is_str=show_default_is_str, default_value=d
            )
            for d in default_value
        )
    elif isinstance(default_value, Enum):
        default_string = str(default_value.value)
    elif inspect.isfunction(default_value):
        default_string = _("(dynamic)")
    elif isinstance(obj, TyperOption) and obj.is_bool_flag and obj.secondary_opts:
        if obj.default:
            if obj.opts:
                default_string = _split_opt(obj.opts[0])[1]
            else:
                default_string = str(default_value)
        else:
            default_string = _split_opt(obj.secondary_opts[0])[1]
    elif (
        isinstance(obj, TyperOption)
        and obj.is_bool_flag
        and not obj.secondary_opts
        and not default_value
    ):
        default_string = ""
    else:
        default_string = str(default_value)
    return default_string


def _extract_default_help_str(
    obj: Union["TyperArgument", "TyperOption"], *, ctx: click.Context
) -> Optional[Union[Any, Callable[[], Any]]]:
    resilient = ctx.resilient_parsing
    ctx.resilient_parsing = True

    try:
        default_value = obj.get_default(ctx, call=False)
    finally:
        ctx.resilient_parsing = resilient
    return default_value


def _handle_exceptions(self: click.Command):
    try:
        self._execute_command()
    except EOFError as e:
        click.echo(file=sys.stderr)
        raise click.Abort() from e
    except KeyboardInterrupt as e:
        raise click.exceptions.Exit(KeyboardInterruptExitCode) from e
    except click.ClickException as e:
        if not self.standalone_mode:
            raise
        rich_utils.rich_format_error(e) if rich and self.rich_markup_mode else e.show()
        sys.exit(e.exit_code)
    except OSError as e:
        if e.errno == errno.EPIPE:
            sys.stdout = cast(TextIO, click.utils.PacifyFlushWrapper(sys.stdout))
            sys.stderr = cast(TextIO, click.utils.PacifyFlushWrapper(sys.stderr))
            sys.exit(1)
        else:
            raise


def _main(
    self: click.Command,
    *,
    args: Optional[Sequence[str]] = None,
    prog_name: Optional[str] = None,
    complete_var: Optional[str] = None,
    standalone_mode: bool = True,
    windows_expand_args: bool = True,
    rich_markup_mode: MarkupMode = DEFAULT_MARKUP_MODE,
    **extra: Any,
) -> Any:
    if args is None:
        args = sys.argv[1:]
        if os.name == "nt" and windows_expand_args:
            args = click.utils._expand_args(args)
    else:
        args = list(args)

    if prog_name is None:
        prog_name = click.utils._detect_program_name()

    self._main_shell_completion(extra, prog_name, complete_var)

    with self.make_context(prog_name, args, **extra) as ctx:
        rv = self.invoke(ctx)
        if not standalone_mode:
            return rv
        ctx.exit()

    _handle_exceptions(self)


# Define constants for exit codes
KeyboardInterruptExitCode = 130

====== [CORRECTED CODE END] =======  
```  

### Explanation of Corrections:
1. **Duplicate Code**: Moved the duplicated `_get_default_string` and `_extract_default_help_str` methods outside the `TyperArgument` and `TyperOption` classes to avoid redundancy.  
2. **Long Method**: Extracted exception handling logic from `_main` into a separate `_handle_exceptions` method to reduce the complexity of the `_main` function.  
3. **Magic Numbers**: Replaced the magic number `130` with a constant `KeyboardInterruptExitCode` for better readability and maintainability.