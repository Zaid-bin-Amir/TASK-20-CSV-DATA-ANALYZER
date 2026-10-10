import csv
import math
import sys
from collections import Counter, defaultdict

INPUT_FILE = "employee_data.csv"
REPORT_FILE = "summary_report.txt"


def parse_number(value):
    if value is None:
        return None
    text = value.strip()
    if not text:
        return None
    try:
        number = float(text)
    except ValueError:
        return None
    if not math.isfinite(number) or number < 0:
        return None
    return number


def load_records(path):
    with open(path, newline="", encoding="utf-8") as file:
        return list(csv.DictReader(file))


def extract_numbers(records, column):
    valid = []
    skipped = 0
    for record in records:
        number = parse_number(record.get(column))
        if number is None:
            skipped += 1
        else:
            valid.append(number)
    return valid, skipped


def column_statistics(records, column):
    values, skipped = extract_numbers(records, column)
    if not values:
        return {
            "column": column,
            "count": 0,
            "skipped": skipped,
            "total": 0.0,
            "average": 0.0,
            "highest": None,
            "lowest": None,
        }
    return {
        "column": column,
        "count": len(values),
        "skipped": skipped,
        "total": sum(values),
        "average": sum(values) / len(values),
        "highest": max(values),
        "lowest": min(values),
    }


def top_record(records, column, label_column):
    best_label = None
    best_value = None
    for record in records:
        number = parse_number(record.get(column))
        if number is None:
            continue
        if best_value is None or number > best_value:
            best_value = number
            best_label = record.get(label_column, "Unknown")
    return best_label, best_value


def count_by_category(records, column):
    counter = Counter()
    for record in records:
        category = (record.get(column) or "").strip() or "Unknown"
        counter[category] += 1
    return dict(counter)


def average_by_category(records, category_column, value_column):
    sums = defaultdict(float)
    counts = defaultdict(int)
    for record in records:
        number = parse_number(record.get(value_column))
        if number is None:
            continue
        category = (record.get(category_column) or "").strip() or "Unknown"
        sums[category] += number
        counts[category] += 1
    return {category: sums[category] / counts[category] for category in sums}


def build_summary(records):
    return {
        "total_records": len(records),
        "salary": column_statistics(records, "salary"),
        "sales": column_statistics(records, "sales"),
        "age": column_statistics(records, "age"),
        "top_earner": top_record(records, "salary", "name"),
        "top_seller": top_record(records, "sales", "name"),
        "department_counts": count_by_category(records, "department"),
        "department_avg_salary": average_by_category(records, "department", "salary"),
    }


def format_statistics(stats):
    if stats["count"] == 0:
        return f"  No valid values (skipped: {stats['skipped']})"
    return "\n".join(
        [
            f"  Valid values : {stats['count']}",
            f"  Skipped      : {stats['skipped']}",
            f"  Total        : {stats['total']:,.2f}",
            f"  Average      : {stats['average']:,.2f}",
            f"  Highest      : {stats['highest']:,.2f}",
            f"  Lowest       : {stats['lowest']:,.2f}",
        ]
    )


def format_top(title, entry):
    label, value = entry
    if label is None:
        return f"{title}: no data"
    return f"{title}: {label} ({value:,.2f})"


def format_report(summary):
    lines = [
        "=" * 50,
        "EMPLOYEE DATA SUMMARY REPORT",
        "=" * 50,
        f"Total records: {summary['total_records']}",
        "",
        "SALARY",
        format_statistics(summary["salary"]),
        "",
        "SALES",
        format_statistics(summary["sales"]),
        "",
        "AGE",
        format_statistics(summary["age"]),
        "",
        format_top("Highest salary", summary["top_earner"]),
        format_top("Highest sales ", summary["top_seller"]),
        "",
        "EMPLOYEES PER DEPARTMENT",
    ]
    for department, count in sorted(summary["department_counts"].items()):
        lines.append(f"  {department:<12} {count}")
    lines.append("")
    lines.append("AVERAGE SALARY PER DEPARTMENT")
    for department, average in sorted(summary["department_avg_salary"].items()):
        lines.append(f"  {department:<12} {average:,.2f}")
    lines.append("=" * 50)
    return "\n".join(lines)


def save_report(text, path):
    with open(path, "w", encoding="utf-8") as file:
        file.write(text + "\n")


def main():
    input_path = sys.argv[1] if len(sys.argv) > 1 else INPUT_FILE
    try:
        records = load_records(input_path)
    except FileNotFoundError:
        print(f"File not found: {input_path}")
        return 1
    if not records:
        print("The CSV file contains no data rows.")
        return 1
    report = format_report(build_summary(records))
    print(report)
    save_report(report, REPORT_FILE)
    print(f"\nReport saved to {REPORT_FILE}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
