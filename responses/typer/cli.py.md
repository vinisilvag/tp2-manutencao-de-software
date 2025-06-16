### Code Smells Detected:

1. **Type**: (`LongMethod`, 57, 95)  
   The method `maybe_update_state` is relatively long and could be broken down into smaller functions for better readability and maintainability.

2. **Type**: (`DuplicatedCode`, 115, 136 and 137, 158)  
   The code blocks in `get_typer_from_module` for iterating over `default_app_names` and `default_func_names` are very similar, leading to code duplication.

3. **Type**: (`MagicNumber`, 211, 213)  
   The `indent` parameter in `get_docs_for_click` uses a magic number (`1 + indent`) which could be confusing without context.

---

### Corrected Code:

====== [CORRECTED CODE START] =======

import importlib.util
import re
import sys
from pathlib import Path
from typing import Any, List, Optional

import click
import typer
import typer.core
from click import Command, Group, Option

from . import __version__

try:
    import rich

    has_rich = True
    from . import rich_utils

except ImportError:  # pragma: no cover
    has_rich = False
    rich = None  # type: ignore

DEFAULT_INDENT_LEVEL = 1  # ====== [ADDED CODE] ======= Add named constant for magic number
default_app_names = ("app", "cli", "main")
default_func_names = ("main", "cli", "app")

app = typer.Typer()
utils_app = typer.Typer(help="Extra utility commands for Typer apps.")
app.add_typer(utils_app, name="utils")


class State:
    def __init__(self) -> None:
        self.app: Optional[str] = None
        self.func: Optional[str] = None
        self.file: Optional[Path] = None
        self.module: Optional[str] = None


state = State()


def _update_file_state(path_or_module: str) -> None:  # ====== [ADDED CODE] ======= Extract file state update
    file_path = Path(path_or_module)
    if file_path.exists() and file_path.is_file():
        state.file = file_path
    else:
        if not re.fullmatch(r"[a-zA-Z_]\w*(\.[a-zA-Z_]\w*)*", path_or_module):
            typer.echo(f"Not a valid file or Python module: {path_or_module}", err=True)
            sys.exit(1)
        state.module = path_or_module


def maybe_update_state(ctx: click.Context) -> None:  # ====== [MODIFIED CODE] ======= Simplified method
    path_or_module = ctx.params.get("path_or_module")
    if path_or_module:
        _update_file_state(path_or_module)
    if ctx.params.get("app"):
        state.app = ctx.params["app"]
    if ctx.params.get("func"):
        state.func = ctx.params["func"]


class TyperCLIGroup(typer.core.TyperGroup):
    def list_commands(self, ctx: click.Context) -> List[str]:
        self.maybe_add_run(ctx)
        return super().list_commands(ctx)

    def get_command(self, ctx: click.Context, name: str) -> Optional[Command]:
        self.maybe_add_run(ctx)
        return super().get_command(ctx, name)

    def invoke(self, ctx: click.Context) -> Any:
        self.maybe_add_run(ctx)
        return super().invoke(ctx)

    def maybe_add_run(self, ctx: click.Context) -> None:
        maybe_update_state(ctx)
        maybe_add_run_to_cli(self)


def _get_typer_from_default_names(module: Any, names: tuple, obj_type: Any) -> Optional[typer.Typer]:  # ====== [ADDED CODE] ======= Extract duplicate logic
    local_names_set = set(dir(module))
    for name in names:
        obj = getattr(module, name, None)
        if isinstance(obj, obj_type):
            return obj
    for name in local_names_set - set(names):
        obj = getattr(module, name)
        if isinstance(obj, obj_type):
            return obj
    return None


def get_typer_from_module(module: Any) -> Optional[typer.Typer]:
    if state.app:
        obj = getattr(module, state.app, None)
        if not isinstance(obj, typer.Typer):
            typer.echo(f"Not a Typer object: --app {state.app}", err=True)
            sys.exit(1)
        return obj
    if state.func:
        func_obj = getattr(module, state.func, None)
        if not callable(func_obj):
            typer.echo(f"Not a function: --func {state.func}", err=True)
            sys.exit(1)
        sub_app = typer.Typer()
        sub_app.command()(func_obj)
        return sub_app
    typer_app = _get_typer_from_default_names(module, default_app_names, typer.Typer)  # ====== [MODIFIED CODE] ======= Reuse function
    if typer_app:
        return typer_app
    function_app = _get_typer_from_default_names(module, default_func_names, type(lambda: None))  # ====== [MODIFIED CODE] ======= Reuse function
    if function_app:
        sub_app = typer.Typer()
        sub_app.command()(function_app)
        return sub_app
    return None


def get_typer_from_state() -> Optional[typer.Typer]:
    spec = None
    if state.file:
        module_name = state.file.name
        spec = importlib.util.spec_from_file_location(module_name, str(state.file))
    elif state.module:
        spec = importlib.util.find_spec(state.module)
    if spec is None:
        if state.file:
            typer.echo(f"Could not import as Python file: {state.file}", err=True)
        else:
            typer.echo(f"Could not import as Python module: {state.module}", err=True)
        sys.exit(1)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)  # type: ignore
    obj = get_typer_from_module(module)
    return obj


def maybe_add_run_to_cli(cli: click.Group) -> None:
    if "run" not in cli.commands:
        if state.file or state.module:
            obj = get_typer_from_state()
            if obj:
                obj._add_completion = False
                click_obj = typer.main.get_command(obj)
                click_obj.name = "run"
                if not click_obj.help:
                    click_obj.help = "Run the provided Typer app."
                cli.add_command(click_obj)


def print_version(ctx: click.Context, param: Option, value: bool) -> None:
    if not value or ctx.resilient_parsing:
        return
    typer.echo(f"Typer version: {__version__}")
    raise typer.Exit()


@app.callback(cls=TyperCLIGroup, no_args_is_help=True)
def callback(
    ctx: typer.Context,
    *,
    path_or_module: str = typer.Argument(None),
    app: str = typer.Option(None, help="The typer app object/variable to use."),
    func: str = typer.Option(None, help="The function to convert to Typer."),
    version: bool = typer.Option(
        False,
        "--version",
        help="Print version and exit.",
        callback=print_version,
    ),
) -> None:
    """
    Run Typer scripts with completion, without having to create a package.

    You probably want to install completion for the typer command:

    $ typer --install-completion

    https://typer.tiangolo.com/
    """
    maybe_update_state(ctx)


def get_docs_for_click(
    *,
    obj: Command,
    ctx: typer.Context,
    indent: int = 0,
    name: str = "",
    call_prefix: str = "",
    title: Optional[str] = None,
) -> str:
    docs = "#" * (DEFAULT_INDENT_LEVEL + indent)  # ====== [MODIFIED CODE] ======= Use named constant
    command_name = name or obj.name
    if call_prefix:
        command_name = f"{call_prefix} {command_name}"
    if not title:
        title = f"`{command_name}`" if command_name else "CLI"
    docs += f" {title}\n\n"
    if obj.help:
        docs += f"{_parse_html(obj.help)}\n\n"
    usage_pieces = obj.collect_usage_pieces(ctx)
    if usage_pieces:
        docs += "**Usage**:\n\n"
        docs += "```console\n"
        docs += "$ "
        if command_name:
            docs += f"{command_name} "
        docs += f"{' '.join(usage_pieces)}\n"
        docs += "```\n\n"
    args = []
    opts = []
    for param in obj.get_params(ctx):
        rv = param.get_help_record(ctx)
        if rv is not None:
            if param.param_type_name == "argument":
                args.append(rv)
            elif param.param_type_name == "option":
                opts.append(rv)
    if args:
        docs += "**Arguments**:\n\n"
        for arg_name, arg_help in args:
            docs += f"* `{arg_name}`"
            if arg_help:
                docs += f": {_parse_html(arg_help)}"
            docs += "\n"
        docs += "\n"
    if opts:
        docs += "**Options**:\n\n"
        for opt_name, opt_help in opts:
            docs += f"* `{opt_name}`"
            if opt_help:
                docs += f": {_parse_html(opt_help)}"
            docs += "\n"
        docs += "\n"
    if obj.epilog:
        docs += f"{obj.epilog}\n\n"
    if isinstance(obj, Group):
        group = obj
        commands = group.list_commands(ctx)
        if commands:
            docs += "**Commands**:\n\n"
            for command in commands:
                command_obj = group.get_command(ctx, command)
                assert command_obj
                docs += f"* `{command_obj.name}`"
                command_help = command_obj.get_short_help_str()
                if command_help:
                    docs += f": {_parse_html(command_help)}"
                docs += "\n"
            docs += "\n"
        for command in commands:
            command_obj = group.get_command(ctx, command)
            assert command_obj
            use_prefix = ""
            if command_name:
                use_prefix += f"{command_name}"
            docs += get_docs_for_click(
                obj=command_obj, ctx=ctx, indent=indent + 1, call_prefix=use_prefix
            )
    return docs


def _parse_html(input_text: str) -> str:
    if not has_rich:  # pragma: no cover
        return input_text
    return rich_utils.rich_to_html(input_text)


@utils_app.command()
def docs(
    ctx: typer.Context,
    name: str = typer.Option("", help="The name of the CLI program to use in docs."),
    output: Optional[Path] = typer.Option(
        None,
        help="An output file to write docs to, like README.md.",
        file_okay=True,
        dir_okay=False,
    ),
    title: Optional[str] = typer.Option(
        None,
        help="The title for the documentation page. If not provided, the name of "
        "the program is used.",
    ),
) -> None:
    """
    Generate Markdown docs for a Typer app.
    """
    typer_obj = get_typer_from_state()
    if not typer_obj:
        typer.echo("No Typer app found", err=True)
        raise typer.Abort()
    click_obj = typer.main.get_command(typer_obj)
    docs = get_docs_for_click(obj=click_obj, ctx=ctx, name=name, title=title)
    clean_docs = f"{docs.strip()}\n"
    if output:
        output.write_text(clean_docs)
        typer.echo(f"Docs saved to: {output}")
    else:
        typer.echo(clean_docs)


def main() -> Any:
    return app()


class UtilityHelper:
    def parse_and_log(self, text: str) -> None:
        parsed = _parse_html(text)
        print(f"Parsed text: {parsed}")

    def write_output(self, file: Path, content: str) -> None:
        file.write_text(content)
        print(f"Wrote to {file}")

====== [CORRECTED CODE END] =======

### Summary of Changes:
1. **Extracted Logic**: Split `maybe_update_state` into a smaller helper function `_update_file_state`.
2. **Removed Duplicate Code**: Created a reusable function `_get_typer_from_default_names` to handle the common logic for iterating over `default_app_names` and `default_func_names`.
3. **Magic Number Replacement**: Replaced the magic number `1 + indent` with a named constant `DEFAULT_INDENT_LEVEL`.