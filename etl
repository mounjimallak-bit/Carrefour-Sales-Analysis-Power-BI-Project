from __future__ import annotations
from pathlib import Path
import pandas as pd
import argparse


OUTPUT_COLUMNS = [
    "order_id",
    "order_date",
    "status",
    "item_id",
    "sku",
    "qty_ordered",
    "price",
    "value",
    "Discount_Percent",
    "discount_amount",
    "category",
    "payment_method",
    "year",
    "month",
    "Gender",
    "age",
    "City",
    "Reasons for cancelation and order refund",
]


def read_csv_smart(path: Path) -> pd.DataFrame:
    for enc in ("utf-8", "utf-8-sig", "cp1252", "latin1"):
        try:
            return pd.read_csv(path, encoding=enc)
        except UnicodeDecodeError:
            continue
    return pd.read_csv(path, encoding="cp1252", encoding_errors="replace")


def strip_text_columns(df: pd.DataFrame) -> pd.DataFrame:
    obj_cols = df.select_dtypes(include=["object"]).columns
    for c in obj_cols:
        df[c] = df[c].astype(str).str.strip()
        df.loc[df[c].str.lower().isin(["nan", "none", "null"]), c] = pd.NA
    return df


def format_order_date(df: pd.DataFrame) -> pd.DataFrame:
    if "order_date" in df.columns:
        dt = pd.to_datetime(df["order_date"], errors="coerce")
        df["order_date"] = dt.dt.strftime("%m/%d/%Y")
    return df


def fix_month_column(df: pd.DataFrame) -> pd.DataFrame:
    if "month" in df.columns:
        if pd.api.types.is_numeric_dtype(df["month"]):
            df["month"] = df["month"].astype("Int64")
            return df

        parsed = pd.to_datetime(df["month"], format="%y-%b", errors="coerce")
        if parsed.notna().mean() < 0.6 and "order_date" in df.columns:
            dt = pd.to_datetime(df["order_date"], errors="coerce")
            df["month"] = dt.dt.month.astype("Int64")
        else:
            df["month"] = parsed.dt.month.astype("Int64")
    else:
        if "order_date" in df.columns:
            dt = pd.to_datetime(df["order_date"], errors="coerce")
            df["month"] = dt.dt.month.astype("Int64")

    return df


def ensure_year(df: pd.DataFrame) -> pd.DataFrame:
    if "year" not in df.columns and "order_date" in df.columns:
        dt = pd.to_datetime(df["order_date"], errors="coerce")
        df["year"] = dt.dt.year.astype("Int64")
    elif "year" in df.columns:
        df["year"] = pd.to_numeric(df["year"], errors="coerce").astype("Int64")
    return df


def fix_discount_amount(df: pd.DataFrame) -> pd.DataFrame:
    if not {"discount_amount", "value", "Discount_Percent"}.issubset(df.columns):
        return df

    df["discount_amount"] = pd.to_numeric(df["discount_amount"], errors="coerce").fillna(0.0)
    df["value"] = pd.to_numeric(df["value"], errors="coerce").fillna(0.0)
    df["Discount_Percent"] = pd.to_numeric(df["Discount_Percent"], errors="coerce").fillna(0.0)

    da100 = df["discount_amount"] * 100.0
    expected = (df["value"] * df["Discount_Percent"] / 100.0) * 100.0

    use_expected = (da100 - expected).abs() > 1e-2
    df["discount_amount"] = da100.where(~use_expected, expected)

    return df


def main(in_path: Path, out_path: Path) -> None:
    df = read_csv_smart(in_path)

    df.columns = [c.strip() for c in df.columns]

    if "Selling price" in df.columns:
        df = df.drop(columns=["Selling price"])

    df = strip_text_columns(df)

    df = format_order_date(df)
    df = fix_month_column(df)
    df = ensure_year(df)

    df = fix_discount_amount(df)

    for c in ["order_id", "item_id", "qty_ordered", "age"]:
        if c in df.columns:
            df[c] = pd.to_numeric(df[c], errors="coerce").astype("Int64")

    for c in ["price", "value", "Discount_Percent", "discount_amount"]:
        if c in df.columns:
            df[c] = pd.to_numeric(df[c], errors="coerce")

    missing = [c for c in OUTPUT_COLUMNS if c not in df.columns]
    if missing:
        raise ValueError(
            "Colonnes manquantes dans Carrefour.csv : "
            + ", ".join(missing)
        )

    df = df[OUTPUT_COLUMNS]

    out_path.parent.mkdir(parents=True, exist_ok=True)
    df.to_csv(out_path, index=False, encoding="cp1252")
    print(f"ETL terminé. Fichier généré: {out_path}")


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Carrefour Sales ETL (raw -> cleaned)")
    parser.add_argument(
        "--input",
        type=str,
        default="Carrefour.csv",
    )
    parser.add_argument(
        "--output",
        type=str,
        default="final_dataset_generated.csv",
    )
    args = parser.parse_args()

    main(Path(args.input), Path(args.output))
