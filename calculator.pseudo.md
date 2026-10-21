# Interactive Console Calculator — Pseudocode Specification

## Overview

This document specifies an interactive console calculator in structured pseudocode; nothing here is compiled, interpreted or executed, and a reader follows it by hand. The calculator runs at a console, one line in and one line out: it prints the prompt `> `, reads one line, prints one line — a result or an error message — and prompts again.

The loop, in order: print the prompt `> `; read one line; trim it and compare it with `quit`, ignoring case, ending the program on a match; otherwise hand the untrimmed line to `parseExpression`, then to `evaluate`, then to `formatResult`, and print what comes back — the result text, or `Error: ` followed by the message text; then prompt again, so the prompt reappears after every result and after every error.

Only `quit` ends the loop — `quit`, `QUIT` and `  quit  ` all end it, and no other line does — while a whitespace-only line falls through to the parser and reports `empty input`.

Declared vocabulary, since no external standard fixes pseudocode syntax: `PROCEDURE`, `RETURN`, `IF/ELSE`, `WHILE`, `PRINT` and the input verb `READ`. `READ` is the one addition to the five keywords the request fixes, because those five contain no input verb while the same request requires a `readLine` procedure that reads one line from the user; it is the only keyword added, so these six are the whole of the vocabulary. Every other form the pseudocode writes is notation and not a keyword: `THEN` opens a tested condition, `END IF`, `END WHILE` and `END PROCEDURE` close their constructs, `:=` assigns a value to a name, a comparison is written `=`, `<` or in words such as `equals`, `is empty`, `ends with` and `is not one of`, `AND` joins two conditions, and `true` is the literal of the loop's always-true test.

No library function is called anywhere in this document, so a reader needs no outside knowledge: the tokenisation rule, the number rule, the operator membership test and the digit rounding and trailing-zero rule are each written out in full below.

Pipeline and its three boundaries: parsing never computes, evaluation never reads and never prints, and `main` is the only procedure that prints. Errors never leave the stage that detects them as output — `parseExpression` and `evaluate` return them as values, and `main` alone turns a returned error value into a printed line.

## Data definitions

- `operand` — one of the two numbers a line supplies; `parseExpression` produces it from operand text and `evaluate` consumes it. A double-precision floating-point value that arrived in one of the accepted textual forms: an optional single leading `-`, one or more digits, and optionally a single `.` with one or more digits after it.
- `operator` — a single-character value, exactly one of `+`, `-`, `*`, `/`. Nothing else reaches `evaluate`, because `parseExpression` rejects every other token before returning.
- `result` — the double-precision value `evaluate` produces and `formatResult` consumes; any value the four operations can produce from two double-precision operands, its only constraint appearing at formatting time.
- `error value` — why a stage could not produce its ordinary result; the second returned value when the status is `error`. Its message text is complete and final when returned — `empty input`, `expected two operands`, `invalid number '<text>'`, `unknown operator '<text>'` or `division by zero` — so no caller rebuilds or rewrites it. It carries built text rather than a bare code because `unknown operator '%'` and `invalid number 'abc'` quote the offending input.

Status convention: `parseExpression` and `evaluate` return a status as their first value, `ok` or `error`. On `ok` the remaining returned values are the two operands and the operator, or the result; on `error` the remaining value is the error value. `main` holds those remaining returned values in one name per call and reads from it only what the status accounts for, under the names this section gives them — the operands and the operator on `ok`, the error value on `error` — so no operand or result name ever carries an error value and no value is read on a path that does not return it. `main` branches on the status alone and never inspects message text to decide what to do.

No state survives a line of input: these four definitions exist only within one iteration of the loop.

## Procedures

### main

`main` is the read-evaluate-print loop and the only procedure that prints; it holds the `Error: ` prefix once, in `errorPrefix`, adds it to every error value it prints, reports no condition of its own, and returns nothing. The quit test runs before parsing, and the untrimmed line is what `main` hands to `parseExpression`. Each stage's remaining returned values are held in one name — `parseValues` for `parseExpression`, `evaluateValues` for `evaluate` — and read under the status convention set out in the Data definitions; the names `main` binds from them are `operand1`, `operator`, `operand2`, `result` and `errorValue`, with the result passed to `formatResult` and the error value printed after the prefix.

```
PROCEDURE main
    errorPrefix := "Error: "
    WHILE true
        PRINT "> "
        lineText := readLine()
        trimmed := lineText with its leading and trailing whitespace removed
        IF trimmed equals "quit", ignoring case THEN
            RETURN
        END IF
        status, parseValues := parseExpression(lineText)
        IF status = "error" THEN
            errorValue := the error value in parseValues
            PRINT errorPrefix followed by errorValue
        ELSE
            operand1, operator, operand2 := the two operands and the operator in parseValues
            status, evaluateValues := evaluate(operand1, operator, operand2)
            IF status = "error" THEN
                errorValue := the error value in evaluateValues
                PRINT errorPrefix followed by errorValue
            ELSE
                result := evaluateValues
                PRINT formatResult(result)
            END IF
        END IF
    END WHILE
END PROCEDURE
```

### readLine

`readLine` takes no arguments, reads one line with `READ`, and returns its text with the line terminator removed, or empty text when the user enters nothing; it has no error value, because reading from the console cannot fail in this product. It neither trims nor judges what it reads, so whitespace-only detection and the empty-line error belong to `parseExpression`.

```
PROCEDURE readLine
    lineText := READ one line from the user
    RETURN lineText with the line terminator removed
END PROCEDURE
```

### parseExpression

`parseExpression` turns one line of text into two operands and an operator, or into an error value; it never computes and never prints, and it calls no procedure, because the tokenisation rule and the number rule are written out here in full.

```
PROCEDURE parseExpression(lineText)
    trimmed := lineText with its leading and trailing whitespace removed
    IF trimmed is empty THEN
        RETURN "error", "empty input"
    END IF
    tokens := the tokens of trimmed, produced by scanning it left to right under the tokenisation rule below
    IF the count of tokens is not three THEN
        RETURN "error", "expected two operands"
    END IF
    IF the first token does not satisfy the number rule below THEN
        RETURN "error", "invalid number '" followed by the first token followed by "'"
    END IF
    IF the third token does not satisfy the number rule below THEN
        RETURN "error", "invalid number '" followed by the third token followed by "'"
    END IF
    IF the second token is not one of "+", "-", "*", "/" THEN
        RETURN "error", "unknown operator '" followed by the second token followed by "'"
    END IF
    RETURN "ok", the double-precision value of the first token, the second token, the double-precision value of the third token
END PROCEDURE
```

**Tokenisation rule.** The scan reads `trimmed` left to right and holds two values: the token list, initially empty, and the position, initially `operand`. Reading past the end of the text gives the empty character, which counts as whitespace in every test of the scan, so every run below stops at the end of the text and the scan is total — every non-space character of `trimmed` ends inside a token, and no line can leave the scan waiting for a step the rule does not give.

- At operand position the token is an optional leading `-` or `+` followed by the maximal run of characters that are neither whitespace nor one of `+ - * /`; when that run is empty the token is the sign alone, and a `*` or `/` — which is not a sign — is a token of its own.
- At operator position a character that is one of `+ - * /` is an operator token on its own; a digit consumes nothing and switches the position to `operand`, which is how `5 3` produces two tokens and therefore `expected two operands` rather than an undefined outcome; any other character begins the maximal run of characters that are neither whitespace nor digits, which is the token, and that is how `%` in `5 % 2` becomes the token `%`.
- A token's kind is the position it was read at: every token formed at operand position — the lone `*` or `/` included, since neither is a sign — is an operand token, and every token formed at operator position is an operator token. After an operand token the position is `operator`, after an operator token it is `operand`, and whitespace between tokens is skipped; that is what makes `2 + 3`, `2+3`, `2+ 3` and `2 +3` all evaluate.

| Line | Tokens the scan produces |
| --- | --- |
| `2 + 3`, `2+3`, `2+ 3`, `2 +3` | `2`, `+`, `3` |
| `-3*-2` | `-3`, `*`, `-2` |
| `abc + 1` | `abc`, `+`, `1` |
| `5 % 2` | `5`, `%`, `2` |
| `*-3` | `*`, `-`, `3` (three tokens — the `*` was read at operand position, so it is an operand token and the position after it is `operator`) |
| `1e3 + 1`, `1,000 + 1`, `2.5.5 + 1` | three tokens each, the whole word being one of them |
| `+3 + 1` | `+3`, `+`, `1` |
| `5 +` | `5`, `+` (two tokens) |
| `5 3` | `5`, `3` (two tokens) |
| `+ +` | `+`, `+` (two tokens) |
| `1 + 2 + 3` | `1`, `+`, `2`, `+`, `3` (five tokens) |
| `- 3 + 1` | `-`, `3`, `+`, `1` (four tokens) |

**Check order.** After the scan, three checks run in this order, and the order is what reproduces the error table exactly: the token count must be three, otherwise `expected two operands`; then the first and then the third token must each satisfy the number rule, otherwise `invalid number '<text>'` naming the first operand that fails; then the second token must be one of `+ - * /`, otherwise `unknown operator '<text>'`. A wrong count is therefore reported first, because the count check precedes the operand checks, so `1e3 + 1 + 2` (five tokens) and `- 3 + 1` (four tokens) never reach the operand check.

**Number rule**, written out in full because a library parse is barred: an optional single leading `-`, then one or more digits, then optionally a single `.` followed by one or more digits, and nothing else; the token's value is a double-precision floating-point value from this point on. Accepted: `3`, `-2.5`, `0.75`. Rejected, each as `invalid number '<text>'` quoting the offending token: a leading `+` (`+3 + 1` gives `invalid number '+3'`), a second `.` (`2.5.5 + 1` gives `invalid number '2.5.5'`), exponent characters (`1e3 + 1` gives `invalid number '1e3'`) and comma separators (`1,000 + 1` gives `invalid number '1,000'`).

### evaluate

`evaluate` applies the operator to the two operands; it never reads input, never prints, and calls no procedure. `+`, `-` and `*` on double-precision values cannot fail and define no error, and a leading minus on either operand is an ordinary value, so `-3 * -2` yields `6`. For `/` the divisor is tested for zero before the division is performed, so the pseudocode never divides by zero; `0 / 5` yields `0`.

```
PROCEDURE evaluate(operand1, operator, operand2)
    IF operator = "/" AND operand2 = 0 THEN
        RETURN "error", "division by zero"
    END IF
    IF operator = "+" THEN
        result := operand1 + operand2
    ELSE IF operator = "-" THEN
        result := operand1 - operand2
    ELSE IF operator = "*" THEN
        result := operand1 * operand2
    ELSE
        result := operand1 / operand2
    END IF
    RETURN "ok", result
END PROCEDURE
```

### formatResult

`formatResult` renders a numeric result as the display text `main` prints, and it is the only place in this specification where a value is rounded; it has no error value, because every double-precision value has a defined rendering. Its contract is three steps in this order: round the magnitude of the value to 10 decimal places using round-half-away-from-zero; render the rounded value with exactly ten fractional digits, so the number of digits is known before any removal; then delete trailing `0` characters and, if the point is left bare, the point itself. Nothing is ever deleted from the integer part, so a whole number prints as `6` and never as an empty string, and a value whose magnitude rounds to zero prints `0`, never `-0`; a carry out of the tenth place propagates leftward into the integer part, so `0.99999999995` renders as `1`.

```
PROCEDURE formatResult(value)
    IF value is not equal to itself THEN
        RETURN "not a number"
    END IF
    sign := ""
    IF value < 0 THEN
        sign := "-"
    END IF
    IF value - value is not zero THEN
        RETURN sign followed by "infinity"
    END IF
    magnitude := the magnitude of value
    rounded := magnitude rounded at the tenth place by the tie rule below
    wholeText := the decimal text of the whole part of rounded, by the digit extraction below
    fractionText := the ten fractional digits of rounded
    resultText := wholeText followed by "." followed by fractionText
    WHILE resultText ends with "0"
        resultText := resultText without its last character
    END WHILE
    IF resultText ends with "." THEN
        resultText := resultText without its last character
    END IF
    IF resultText = "0" THEN
        RETURN "0"
    END IF
    RETURN sign followed by resultText
END PROCEDURE
```

**Producing the digits without a library call.** The whole part yields its digits by repeated division by 10: take the whole quotient as the next iteration's value and the remainder as the next digit, until the quotient is zero — the extraction always emits at least one digit, so a whole part of zero is the text `0`, and the digits taken are read in the reverse of the order in which they were produced, so a whole part of `10` is the text `10` and never `01`. The fractional part yields its digits one decimal place at a time, up to ten: multiply the remaining fraction by 10 and take the whole part as the next digit. Both loops use only the four operators this pseudocode already has, and the remainder left after the tenth digit is what the tie rule compares against half — when it is at least half, 1 is added to the ten digits and carries into the whole part if the ten digits overflow, so the tie rule yields `rounded`, the value both extractions above are taken from. Under that rule `0.125` rounded to two decimal places is `0.13`.

**Non-finite results.** An infinity renders as `infinity`, a negative infinity as `-infinity` and a not-a-number result as `not a number`, and `main` prints that text exactly as it prints any other result — these are results, not errors, so no `Error: ` prefix applies and no error condition is added. The branch recognises such a value without a library call: a value that is not equal to itself is the not-a-number case, and among the values equal to themselves one whose difference from itself is not zero is an infinity, whose sign comes from comparing it with zero; every finite value is equal to itself and subtracts to zero, so it takes neither branch.

**The magnitude boundary.** Ten fractional digits are meaningful only while the value and the arithmetic that produced it have digits to spare: a double carries 15 to 17 significant decimal digits, so a magnitude beyond about 10^5 leaves fewer than ten decimal places of significance, and beyond about 9 × 10^15 not every whole number is representable — the spacing between neighbouring doubles is 16 at 10^17 and 128 at 10^18 — so the digits of a large value reflect that spacing. Rendering itself does not degrade: a power of ten is exactly representable, so 10^17 renders as `100000000000000000` with ten zero fractional digits to strip. Every acceptance row in the Worked examples section below sits well inside the range where the ten digits are meaningful, the largest computed value being `6`.

**Rounding happens once**, at the tenth fractional place: the two extraction loops only read digits and never round, and intermediate values stay double-precision, so this specification adds no cumulative error of its own.

**Worked checks.** `2 + 3` gives `5`; `10 - 4.5` gives `5.5`; `7 / 2` gives `3.5`; `1 / 3` gives `0.3333333333`; `-3 * -2` gives `6`; `0 / 5` gives `0`.

## Error messages

| Condition | Example | Output | Owning branch |
| --- | --- | --- | --- |
| Empty or whitespace-only line | (empty line) | `Error: empty input` | The emptiness check after trimming, in `parseExpression` |
| Missing operand | `5 +` | `Error: expected two operands` | The token-count check in `parseExpression` |
| More than two operands | `1 + 2 + 3` | `Error: expected two operands` | The same token-count check — one branch, one message, two rows |
| Non-numeric operand | `abc + 1` | `Error: invalid number 'abc'` | The number-rule check on operand 1, then operand 2, in `parseExpression` |
| Unknown operator | `5 % 2` | `Error: unknown operator '%'` | The operator-membership check in `parseExpression` |
| Division by zero | `5 / 0` | `Error: division by zero` | The zero-divisor test in `evaluate` |

The message text is used character for character, including the quoted offending text in `unknown operator '%'` and `invalid number 'abc'`. The `Error: ` prefix is `main`'s alone: it is held once, in `errorPrefix`, and written before the error value rather than into any message, so no message text carries the prefix and a future change to the prefix is a single edit.

Each case prints its message and returns to the prompt, and no case ends the program.

Ownership: `parseExpression` owns four messages, `evaluate` owns the fifth (`division by zero`), and `main` owns the prefix and the printing and reports no condition of its own — no two stages claim the same failure, and no condition can be reported twice.

## Worked examples

Every acceptance row of this document — the eleven inputs listed below, each with the output its own trace states — is traced through the stages; the `quit` row traces the loop's exit test instead, because it never reaches the parser.

- `2 + 3` — tokens `2`, `+`, `3`; `parseExpression` returns status `ok`, operand 1 `2`, operator `+`, operand 2 `3`; `evaluate` applies `+` and returns result `5`; `formatResult` renders `5.0000000000` and strips the trailing zeros and the bare point to `5`; `main` prints `5`.
- `10 - 4.5` — tokens `10`, `-`, `4.5`; status `ok`, operand 1 `10`, operator `-`, operand 2 `4.5`; `evaluate` applies `-`, result `5.5`; `formatResult` strips `5.5000000000` to `5.5`; `main` prints `5.5`.
- `-3 * -2` — tokens `-3`, `*`, `-2`; status `ok`, operand 1 `-3`, operator `*`, operand 2 `-2`; `evaluate` applies `*`, result `6`; `formatResult` strips `6.0000000000` to `6`; `main` prints `6`.
- `7 / 2` — tokens `7`, `/`, `2`; status `ok`, operand 1 `7`, operator `/`, operand 2 `2`; `evaluate` applies `/`, result `3.5`; `formatResult` strips `3.5000000000` to `3.5`; `main` prints `3.5`.
- `1 / 3` — tokens `1`, `/`, `3`; status `ok`, operand 1 `1`, operator `/`, operand 2 `3`; `evaluate` applies `/`, result the double-precision value closest to one third; `formatResult` takes its ten fractional digits `3333333333`, the remainder after the tenth digit being below half so nothing is added, giving `0.3333333333`; `main` prints `0.3333333333`.
- `5 / 0` — tokens `5`, `/`, `0`; status `ok`, operand 1 `5`, operator `/`, operand 2 `0`; `evaluate` finds the divisor zero and returns status `error` with `division by zero`; the trace stops in `evaluate`, and `formatResult` is never reached; `main` prints `Error: division by zero`.
- `5 % 2` — tokens `5`, `%`, `2`, three of them with both operands valid, and the middle token is not one of the four, so `parseExpression` returns status `error` with `unknown operator '%'`; the trace stops in `parseExpression`, before `evaluate`, and `formatResult` is never reached; `main` prints `Error: unknown operator '%'`.
- `abc + 1` — tokens `abc`, `+`, `1`, three of them, and operand 1 fails the number rule, so `parseExpression` returns status `error` with `invalid number 'abc'`; the trace stops in `parseExpression`, before `evaluate`, and `formatResult` is never reached; `main` prints `Error: invalid number 'abc'`.
- `1 + 2 + 3` — tokens `1`, `+`, `2`, `+`, `3`, five of them, and the count check runs first, so `parseExpression` returns status `error` with `expected two operands`; the trace stops in `parseExpression`, before `evaluate`, and `formatResult` is never reached; `main` prints `Error: expected two operands`.
- (empty line) — the trimmed line is empty, so `parseExpression` returns status `error` with `empty input`; the trace stops in `parseExpression`, before `evaluate`, and `formatResult` is never reached; `main` prints `Error: empty input`. A whitespace-only line behaves in the same way.
- `quit` — the trimmed line equals `quit` ignoring case, so `main` returns before parsing and the program ends. `QUIT` and `  quit  ` behave in the same way.

Verification is by hand-trace: a reviewer follows the pseudocode on every worked example and on at least three inputs of their own — one valid and two invalid — and confirms the stated output. No automated check is part of acceptance.
