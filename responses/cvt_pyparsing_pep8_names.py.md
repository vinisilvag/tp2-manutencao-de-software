Code Smells Detected:
1. **Type**: (MagicNumber, 98, 98)
   - The exit status code `1` is a magic number. It should be replaced with a named constant for better readability and maintainability.

2. **Type**: (LongMethod, 70, 136)
   - The `main` block is too long and performs multiple responsibilities: argument parsing, file processing, and diff showing. It should be broken into smaller functions.

3. **Type**: (DuplicateCode, 93, 119)
   - The logic for handling file processing and diffs is duplicated. This can be refactored into a separate function.

Corrected Code
====== [CORRECTED CODE START] =======
from functools import lru_cache
import pyparsing as pp
import argparse
from pathlib import Path
import sys
import difflib


@lru_cache(maxsize=None)
def camel_to_snake(s: str) -> str:
    """
    Convert CamelCase to snake_case.
    """
    return "".join("_" + c.lower() if c.isupper() else c for c in s).lstrip("_")


pre_pep8_method_names = """
addCondition addParseAction anyCloseTag anyOpenTag asDict asList cStyleComment canParseNext conditionAsParseAction 
convertToDate convertToDatetime convertToFloat convertToInteger countedArray cppStyleComment dblQuotedString 
dblSlashComment defaultName dictOf disableMemoization downcaseTokens enableLeftRecursion enablePackrat getName 
htmlComment ignoreWhitespace indentedBlock infixNotation inlineLiteralsUsing javaStyleComment leaveWhitespace 
lineEnd lineStart locatedExpr matchOnlyAtCol matchPreviousExpr matchPreviousLiteral nestedExpr nullDebugAction oneOf 
originalTextFor parseFile parseString parseWithTabs pythonStyleComment quotedString removeQuotes replaceWith 
resetCache restOfLine runTests scanString searchString setBreak setDebug setDebugActions setDefaultWhitespaceChars 
setFailAction setName setParseAction setResultsName setWhitespaceChars sglQuotedString stringEnd stringStart tokenMap 
traceParseAction transformString tryParse unicodeString upcaseTokens withAttribute withClass
""".split()

special_changes = {
    "opAssoc": "OpAssoc",
    "delimitedList": "DelimitedList",
    "delimited_list": "DelimitedList",
    "replaceHTMLEntity": "replace_html_entity",
    "makeHTMLTags": "make_html_tags",
    "makeXMLTags": "make_xml_tags",
    "commonHTMLEntity": "common_html_entity",
    "stripHTMLTags": "strip_html_tags",
}

pre_pep8_arg_names = """parseAll maxMatches listAllMatches callDuringTry includeSeparators fullDump printResults 
failureTests postParse matchString identChars maxMismatches initChars bodyChars asKeyword excludeChars asGroupList 
asMatch quoteChar escChar escQuote unquoteResults endQuoteChar convertWhitespaceEscapes notChars wordChars stopOn 
failOn joinString markerString intExpr useRegex asString ignoreExpr""".split()

pre_pep8_method_name = pp.one_of(pre_pep8_method_names, as_keyword=True)
pre_pep8_method_name.set_parse_action(lambda t: camel_to_snake(t[0]))
special_pre_pep8_name = pp.one_of(special_changes, as_keyword=True)
special_pre_pep8_name.set_parse_action(lambda t: special_changes[t[0]])
# only replace arg names if part of an arg list
pre_pep8_arg_name = pp.Regex(
    rf"{pp.util.make_compressed_re(pre_pep8_arg_names)}\s*="
)
pre_pep8_arg_name.set_parse_action(lambda t: camel_to_snake(t[0]))

pep8_converter = pre_pep8_method_name | special_pre_pep8_name | pre_pep8_arg_name

# ====== [ADDED CODE] =======
CHANGES_MADE_EXIT_CODE = 1

def show_diffs(original, modified):
    diff = difflib.unified_diff(
        original.splitlines(), modified.splitlines(), lineterm=""
    )
    sys.stdout.writelines(f"{diff_line}\n" for diff_line in diff)

def process_file(filename, encoding, update, verbose):
    try:
        original_contents = Path(filename).read_text(encoding=encoding)
        modified_contents = pep8_converter.transform_string(original_contents)

        if modified_contents != original_contents:
            if update:
                Path(filename).write_text(modified_contents, encoding=encoding)
                print(f"Converted {filename}")
            else:
                print(f"Found required changes in {filename}")

            if verbose:
                show_diffs(original_contents, modified_contents)
                print()

            return CHANGES_MADE_EXIT_CODE
        else:
            if verbose:
                print(f"No required changes in {filename}")
        return 0
    except Exception as e:
        print(f"Failed to convert {filename}: {type(e).__name__}: {e}")
        return 0

def main():
    argparser = argparse.ArgumentParser(
        description = (
            "Utility to convert Python pyparsing scripts using legacy"
            " camelCase names to use PEP8 snake_case names."
            "\nBy default, this script will only show whether this script would make any changes."
        )
    )
    argparser.add_argument("--verbose", "-v", action="store_true", help="Show unified diff for each source file")
    argparser.add_argument("-vv", action="store_true", dest="verbose2", help="Show unified diff for each source file, plus names of scanned files with no changes")
    argparser.add_argument("--update", "-u", action="store_true", help="Update source files in-place")
    argparser.add_argument("--encoding", type=str, default="utf-8", help="Encoding of source files (default: utf-8)")
    argparser.add_argument("--exit-zero-even-if-changed", "-exit0", action="store_true", help="Exit with status code 0 even if changes were made")
    argparser.add_argument("source_filename", nargs="+", help="Source filenames or filename patterns of Python files to be converted")
    args = argparser.parse_args()

    exit_status = 0

    for filename_pattern in args.source_filename:
        for filename in Path().glob(filename_pattern):
            if not Path(filename).is_file():
                continue
            exit_status |= process_file(filename, args.encoding, args.update, args.verbose or args.verbose2)

    sys.exit(exit_status if not args.exit_zero_even_if_changed else 0)

if __name__ == "__main__":
    main()
====== [CORRECTED CODE END] =======
```

### Explanation of Refactorings:
1. **MagicNumber**: Replaced the exit status code `1` with a named constant `CHANGES_MADE_EXIT_CODE` for better clarity.
2. **LongMethod**: The main block was divided into smaller functions: `show_diffs`, `process_file`, and `main` to improve readability and maintainability.
3. **DuplicateCode**: The file processing logic was moved into the `process_file` function to eliminate duplication.

These changes make the code more modular, easier to understand, and maintain.