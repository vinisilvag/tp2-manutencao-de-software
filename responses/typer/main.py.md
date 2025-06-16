### Code Smells Detected:
1. **Type**: (Long Method, 37, 78)  
   - **Description**: The `except_hook` function is too long and handles multiple responsibilities, including traceback formatting and exception handling logic.

2. **Type**: (Duplicated Code, 685, 710)  
   - **Description**: The `launch` function contains duplicated code for handling URL launching in different operating systems.

3. **Type**: (Magic Numbers, 48, 48)  
   - **Description**: The `MAX_WIDTH` is used without a clear explanation of its purpose or origin.

4. **Type**: (Long Method, 109, 161)  
   - **Description**: The `__init__` method of the `Typer` class is too long and initializes a large number of attributes, making it hard to maintain.

### Corrected Code:

```python
import inspect
import os
import platform
import shutil
import subprocess
import sys
import traceback
from datetime import datetime
from enum import Enum
from functools import update_wrapper
from pathlib import Path
from traceback import FrameSummary, StackSummary
from types import TracebackType
from typing import Any, Callable, Dict, List, Optional, Sequence, Tuple, Type, Union
from uuid import UUID

import click
from typer._types import TyperChoice

from ._typing import get_args, get_origin, is_union
from .completion import get_completion_inspect_parameters
from .core import (
    DEFAULT_MARKUP_MODE,
    MarkupMode,
    TyperArgument,
    TyperCommand,
    TyperGroup,
    TyperOption,
)
from .models import (
    AnyType,
    ArgumentInfo,
    CommandFunctionType,
    CommandInfo,
    Default,
    DefaultPlaceholder,
    DeveloperExceptionConfig,
    FileBinaryRead,
    FileBinaryWrite,
    FileText,
    FileTextWrite,
    NoneType,
    OptionInfo,
    ParameterInfo,
    ParamMeta,
    Required,
    TyperInfo,
    TyperPath,
)
from .utils import get_params_from_function

try:
    import rich
    from rich.traceback import Traceback

    from . import rich_utils

    console_stderr = rich_utils._get_rich_console(stderr=True)

except ImportError:  # pragma: no cover
    rich = None  # type: ignore


_original_except_hook = sys.excepthook
_typer_developer_exception_attr_name = "__typer_developer_exception__"

def _format_rich_traceback(exc, exception_config, typer_path, click_path):
    from .rich_utils import MAX_WIDTH
    rich_tb = Traceback.from_exception(
        type(exc),
        exc,
        exc.__traceback__,
        show_locals=exception_config.pretty_exceptions_show_locals,
        suppress=[typer_path, click_path],
        width=MAX_WIDTH,
    )
    console_stderr.print(rich_tb)

def _format_standard_traceback(exc, exception_config, typer_path, click_path):
    tb_exc = traceback.TracebackException.from_exception(exc)
    stack: List[FrameSummary] = []
    for frame in tb_exc.stack:
        if any(frame.filename.startswith(path) for path in [typer_path, click_path]):
            if not exception_config.pretty_exceptions_short:
                stack.append(
                    traceback.FrameSummary(
                        filename=frame.filename,
                        lineno=frame.lineno,
                        name=frame.name,
                        line="",
                    )
                )
        else:
            stack.append(frame)
    final_stack_summary = StackSummary.from_list(stack)
    tb_exc.stack = final_stack_summary
    for line in tb_exc.format():
        print(line, file=sys.stderr)

def except_hook(
    exc_type: Type[BaseException], exc_value: BaseException, tb: Optional[TracebackType]
) -> None:
    exception_config: Union[DeveloperExceptionConfig, None] = getattr(
        exc_value, _typer_developer_exception_attr_name, None
    )
    standard_traceback = os.getenv("_TYPER_STANDARD_TRACEBACK")
    if (
        standard_traceback
        or not exception_config
        or not exception_config.pretty_exceptions_enable
    ):
        _original_except_hook(exc_type, exc_value, tb)
        return
    typer_path = os.path.dirname(__file__)
    click_path = os.path.dirname(click.__file__)
    exc = exc_value
    if rich:
        _format_rich_traceback(exc, exception_config, typer_path, click_path)
    else:
        _format_standard_traceback(exc, exception_config, typer_path, click_path)
    return

def get_install_completion_arguments() -> Tuple[click.Parameter, click.Parameter]:
    install_param, show_param = get_completion_inspect_parameters()
    click_install_param, _ = get_click_param(install_param)
    click_show_param, _ = get_click_param(show_param)
    return click_install_param, click_show_param

class Typer:
    def __init__(
        self,
        *,
        name: Optional[str] = Default(None),
        cls: Optional[Type[TyperGroup]] = Default(None),
        invoke_without_command: bool = Default(False),
        no_args_is_help: bool = Default(False),
        subcommand_metavar: Optional[str] = Default(None),
        chain: bool = Default(False),
        result_callback: Optional[Callable[..., Any]] = Default(None),
        # Command
        context_settings: Optional[Dict[Any, Any]] = Default(None),
        callback: Optional[Callable[..., Any]] = Default(None),
        help: Optional[str] = Default(None),
        epilog: Optional[str] = Default(None),
        short_help: Optional[str] = Default(None),
        options_metavar: str = Default("[OPTIONS]"),
        add_help_option: bool = Default(True),
        hidden: bool = (False),
        deprecated: bool = Default(False),
        add_completion: bool = True,
        # Rich settings
        rich_markup_mode: MarkupMode = Default(DEFAULT_MARKUP_MODE),
        rich_help_panel: Union[str, None] = Default(None),
        pretty_exceptions_enable: bool = True,
        pretty_exceptions_show_locals: bool = True,
        pretty_exceptions_short: bool = True,
    ):
        self._add_completion = add_completion
        self.rich_markup_mode: MarkupMode = rich_markup_mode
        self.rich_help_panel = rich_help_panel
        self.pretty_exceptions_enable = pretty_exceptions_enable
        self.pretty_exceptions_show_locals = pretty_exceptions_show_locals
        self.pretty_exceptions_short = pretty_exceptions_short
        self.info = TyperInfo(
            name=name,
            cls=cls,
            invoke_without_command=invoke_without_command,
            no_args_is_help=no_args_is_help,
            subcommand_metavar=subcommand_metavar,
            chain=chain,
            result_callback=result_callback,
            context_settings=context_settings,
            callback=callback,
            help=help,
            epilog=epilog,
            short_help=short_help,
            options_metavar=options_metavar,
            add_help_option=add_help_option,
            hidden=hidden,
            deprecated=deprecated,
        )
        self.registered_groups: List[TyperInfo] = []
        self.registered_commands: List[CommandInfo] = []
        self.registered_callback: Optional[TyperInfo] = None

    def callback(
        self,
        *,
        cls: Optional[Type[TyperGroup]] = Default(None),
        invoke_without_command: bool = Default(False),
        no_args_is_help: bool = Default(False),
        subcommand_metavar: Optional[str] = Default(None),
        chain: bool = Default(False),
        result_callback: Optional[Callable[..., Any]] = Default(None),
        # Command
        context_settings: Optional[Dict[Any, Any]] = Default(None),
        help: Optional[str] = Default(None),
        epilog: Optional[str] = Default(None),
        short_help: Optional[str] = Default(None),
        options_metavar: str = Default("[OPTIONS]"),
        add_help_option: bool = Default(True),
        hidden: bool = Default(False),
        deprecated: bool = Default(False),
        # Rich settings
        rich_help_panel: Union[str, None] = Default(None),
    ) -> Callable[[CommandFunctionType], CommandFunctionType]:
        def decorator(f: CommandFunctionType) -> CommandFunctionType:
            self.registered_callback = TyperInfo(
                cls=cls,
                invoke_without_command=invoke_without_command,
                no_args_is_help=no_args_is_help,
                subcommand_metavar=subcommand_metavar,
                chain=chain,
                result_callback=result_callback,
                context_settings=context_settings,
                callback=f,
                help=help,
                epilog=epilog,
                short_help=short_help,
                options_metavar=options_metavar,
                add_help_option=add_help_option,
                hidden=hidden,
                deprecated=deprecated,
                rich_help_panel=rich_help_panel,
            )
            return f

        return decorator

    def command(
        self,
        name: Optional[str] = None,
        *,
        cls: Optional[Type[TyperCommand]] = None,
        context_settings: Optional[Dict[Any, Any]] = None,
        help: Optional[str] = None,
        epilog: Optional[str] = None,
        short_help: Optional[str] = None,
        options_metavar: str = "[OPTIONS]",
        add_help_option: bool = True,
        no_args_is_help: bool = False,
        hidden: bool = False,
        deprecated: bool = False,
        # Rich settings
        rich_help_panel: Union[str, None] = Default(None),
    ) -> Callable[[CommandFunctionType], CommandFunctionType]:
        if cls is None:
            cls = TyperCommand

        def decorator(f: CommandFunctionType) -> CommandFunctionType:
            self.registered_commands.append(
                CommandInfo(
                    name=name,
                    cls=cls,
                    context_settings=context_settings,
                    callback=f,
                    help=help,
                    epilog=epilog,
                    short_help=short_help,
                    options_metavar=options_metavar,
                    add_help_option=add_help_option,
                    no_args_is_help=no_args_is_help,
                    hidden=hidden,
                    deprecated=deprecated,
                    # Rich settings
                    rich_help_panel=rich_help_panel,
                )
            )
            return f

        return decorator

    def add_typer(
        self,
        typer_instance: "Typer",
        *,
        name: Optional[str] = Default(None),
        cls: Optional[Type[TyperGroup]] = Default(None),
        invoke_without_command: bool = Default(False),
        no_args_is_help: bool = Default(False),
        subcommand_metavar: Optional[str] = Default(None),
        chain: bool = Default(False),
        result_callback: Optional[Callable[..., Any]] = Default(None),
        # Command
        context_settings: Optional[Dict[Any, Any]] = Default(N0one),
        callback: Optional[Callable[..., Any]] = Default(None),
        help: Optional[str] = Default(None),
        epilog: Optional[str] = Default(None),
        short_help: Optional[str] = Default(None),
        options_metavar: str = Default("[OPTIONS]"),
        add_help_option: bool = Default(True),
        hidden: bool = Default(False),
        deprecated: bool = Default(False),
        # Rich settings
        rich_help_panel: Union[str, None] = Default(None),
    ) -> None:
        self.registered_groups.append(
            TyperInfo(
                typer_instance,
                name=name,
                cls=cls,
                invoke_without_command=invoke_without_command,
                no_args_is_help=no_args_is_help,
                subcommand_metavar=subcommand_metavar,
                chain=chain,
                result_callback=result_callback,
                context_settings=context_settings,
                callback=callback,
                help=help,
                epilog=epilog,
                short_help=short_help,
                options_metavar=options_metavar,
                add_help_option=add_help_option,
                hidden=hidden,
                deprecated=deprecated,
                rich_help_panel=rich_help_panel,
            )
        )

    def __call__(self, *args: Any, **kwargs: Any) -> Any:
        if sys.excepthook != except_hook:
            sys.excepthook = except_hook
        try:
            return get_command(self)(*args, **kwargs)
        except Exception as e:
            # Set a custom attribute to tell the hook to show nice exceptions for user
            # code. An alternative/first implementation was a custom exception with
            # raise custom_exc from e
            # but that means the last error shown is the custom exception, not the
            # actual error. This trick improves developer experience by showing the
            # actual error last.
            setattr(
                e,
                _typer_developer_exception_attr_name,
                DeveloperExceptionConfig(
                    pretty_exceptions_enable=self.pretty_exceptions_enable,
                    pretty_exceptions_show_locals=self.pretty_exceptions_show_locals,
                    pretty_exceptions_short=self.pretty_exceptions_short,
                ),
            )
            raise e


def get_group(typer_instance: Typer) -> TyperGroup:
    group = get_group_from_info(
        TyperInfo(typer_instance),
        pretty_exceptions_short=typer_instance.pretty_exceptions_short,
        rich_markup_mode=typer_instance.rich_markup_mode,
    )
    return group


def get_command(typer_instance: Typer) = click.Command:
    if typer_instance._ad_completion:
        click_install_param, click_show_param = get_install_completion_arguments()
    if (
        typer_instance.registered_callback
        or typer_instance.info.callback
        or typer_instance.registered_groups
        or len(typer_instance.registered_commands) > 1
    ):
        # Create a Group
        click_command: click.Command = get_group(typer_instance)
        if typer_instance._add_completion:
            click_command.params.append(click_install_param)
            click_command.params.append(click_show_param)
        return click_command
    elif len(typer_instance.registered_commands) == 1:
        # Create a single Command
        single_command = typer_instance.registered_commands[0]

        if not single_command.context_settings and not isinstance(
            typer_instance.info.context_settings, DefaultPlaceholder
        ):
            single_command.context_settings = typer_instance.info.context_settings

        click_command = get_command_from_info(
            single_command,
            pretty_exceptions_short=typer_instance.pretty_exceptions_short,
            rich_markup_mode=typer_instance.rich_markup_mode,
        )
        if typer_instance._add_completion:
            click_command.params.append(click_install_param)
            click_command.params.append(click_show_param)
        return click_command
    raise RuntimeError(
        "Could not get a command for this Typer instance"
    )  # pragma: no cover

====== [ADDED CODE] =======
def _launch_url_on_macos(url, wait):
    return subprocess.Popen(
        ["open", url], stdout=subprocess.DEVNULL, stderr=subprocess.STDOUT
    ).wait()

def _launch_url_on_linux_or_bsd(url, wait):
    return subprocess.Popen(
        ["xdg-open", url], stdout=subprocess.DEVNULL, stderr=subprocess.STDOUT
    ).wait()

def _launch_url_default(url, wait):
    import webbrowser
    webbrowser.open(url)
    return 0

def launch(url: str, wait: bool = False, locate: bool = False) -> int:
    if url.startswith("http://") or url.startswith("https://"):
        if _is_macos():
            return _launch_url_on_macos(url, wait)
        elif _is_linux_or_bsd() and shutil.which("xdg-open") is not None:
            return _launch_url_on_linux_or_bsd(url, wait)
        else:
            return _launch_url_default(url, wait)
    else:
        return click.launch(url)
====== [CORRECTED CODE END] =======

```