
import sqlite3
from dataclasses import dataclass
import pandas as pd
import numpy as np
import streamlit as st
from io import BytesIO
import calendar
import datetime as dt

def surname_key(full_name: str) -> str:
    """Return surname (last token) for sorting; fallback to full_name."""
    if not full_name:
        return ""
    parts = str(full_name).strip().split()
    return (parts[-1] if parts else str(full_name)).lower()




def split_name_first_surname(full_name: str) -> tuple[str, str]:
    """Assumes stored input like 'Jméno Příjmení ...'. Returns (first_name, surname_part)."""
    if not full_name:
        return ("", "")
    parts = str(full_name).strip().split()
    if len(parts) == 1:
        return (parts[0], "")
    first = parts[0]
    surname = " ".join(parts[1:])
    return (first, surname)

def _title_words(s: str) -> str:
    """Title-case each word, preserving diacritics; handles ALLCAPS surnames."""
    return " ".join([w[:1].upper() + w[1:].lower() if w else "" for w in str(s).strip().split()])

def format_name_surname_first(full_name: str) -> str:
    """Formats display as 'Příjmení Jméno' (e.g., 'Kopp Tomáš')."""
    first, surname = split_name_first_surname(full_name)
    first_t = _title_words(first)
    surname_t = _title_words(surname) if surname else ""
    if surname_t:
        return f"{surname_t} {first_t}".strip()
    return first_t

def _strip_diacritics(s: str) -> str:
    import unicodedata as _ud
    return "".join(ch for ch in _ud.normalize("NFKD", str(s)) if not _ud.combining(ch))

def sort_df_by_surname(df: pd.DataFrame, name_col: str = "Jméno") -> pd.DataFrame:
    """Sort by surname (everything before last token in display 'Příjmení Jméno' is already surname-first),
    using diacritics-insensitive Czech-friendly key."""
    if name_col not in df.columns or df.empty:
        return df
    tmp = df.copy()
    key = tmp[name_col].astype(str).map(lambda s: _strip_diacritics(s).lower())
    tmp = tmp.assign(_k=key).sort_values(by=["_k"], kind="mergesort").drop(columns=["_k"]).reset_index(drop=True)
    return tmp



DB_PATH = "mzdy_app.sqlite"
st.set_page_config(page_title="Výpočet mezd a prémií (AKENG)", layout="wide")

def get_conn():
    return sqlite3.connect(DB_PATH, check_same_thread=False)

def df_query(q, params=()):
    with get_conn() as conn:
        return pd.read_sql_query(q, conn, params=params)

def exec_query(q, params=()):
    with get_conn() as conn:
        conn.execute(q, params)
        conn.commit()

def table_columns(table_name: str):
    with get_conn() as conn:
        cur = conn.cursor()
        cur.execute(f"PRAGMA table_info({table_name})")
        return {row[1] for row in cur.fetchall()}

def add_column_if_missing(table: str, col: str, coltype: str, default_sql=None):
    cols = table_columns(table)
    if col in cols:
        return
    stmt = f"ALTER TABLE {table} ADD COLUMN {col} {coltype}"
    if default_sql is not None:
        stmt += f" DEFAULT {default_sql}"
    exec_query(stmt)

# --- Migrations ---
add_column_if_missing("employees", "hourly_rate", "REAL", "0")
add_column_if_missing("employees", "hourly_rate_avg", "REAL", "0")  # průměrná hodinovka pro dovolené/svátky
add_column_if_missing("employees", "allow_extra_bonus", "INTEGER", "0")
add_column_if_missing("employees", "avg_hourly_rate", "REAL", "0")  # prumerna hodinova mzda pro nahrady


# --- Auto-apply role updates per směrnice (safe) ---
try:
    exec_query("UPDATE employees SET role=?, receives_prod_bonus=0, counts_to_normative=0, coeff=1.0, allow_extra_bonus=1 WHERE full_name LIKE ?", ("Nákupčí materiálu", "%Ambrozek%Jakub%"))
    exec_query("UPDATE employees SET role=?, receives_prod_bonus=0, counts_to_normative=0, coeff=1.0, allow_extra_bonus=1 WHERE full_name LIKE ?", ("Technolog", "%Obručník%Jan%"))
except Exception:
    pass

add_column_if_missing("attendance", "hours_worked", "REAL", "0")        # pravidelné hodiny (bez přesčasů)
add_column_if_missing("attendance", "overtime_hours", "REAL", "0")      # přesčasové hodiny (navíc)
add_column_if_missing("attendance", "overtime_comp_hours", "REAL", "0") # přesčas jako náhradní volno (hodiny)
add_column_if_missing("attendance", "afternoon_hours", "REAL", "0")
add_column_if_missing("attendance", "holiday_hours", "REAL", "0")
add_column_if_missing("attendance", "holiday_days", "REAL", "0")  # svatky (dny)
add_column_if_missing("attendance", "vacation_days", "REAL", "0")
add_column_if_missing("attendance", "sick_days", "REAL", "0")
add_column_if_missing("attendance", "unpaid_days", "REAL", "0")

exec_query("""
CREATE TABLE IF NOT EXISTS extra_bonus (
  month_key TEXT NOT NULL,
  employee_id INTEGER NOT NULL,
  pct REAL NOT NULL DEFAULT 0,
  PRIMARY KEY (month_key, employee_id),
  FOREIGN KEY(employee_id) REFERENCES employees(id)
)
""")

exec_query("""
CREATE TABLE IF NOT EXISTS evaluations (
  month_key TEXT NOT NULL,
  employee_id INTEGER NOT NULL,
  area_code TEXT NOT NULL,
  score REAL NOT NULL,
  PRIMARY KEY (month_key, employee_id, area_code),
  FOREIGN KEY(employee_id) REFERENCES employees(id)
)
""")

# months defaults
add_column_if_missing("months", "afternoon_pct", "REAL", "0.10")
add_column_if_missing("months", "holiday_pct", "REAL", "1.00")
add_column_if_missing("months", "overtime_pct", "REAL", "0.25")
add_column_if_missing("months", "hours_per_day", "REAL", "8")
add_column_if_missing("months", "holiday_days", "REAL", "0")  # svatky v mesici (pracovni dny) pro vypocet fondu


# ---------------- Upserts ----------------
def upsert_attendance(month_key, employee_id, hours_worked, overtime_hours, overtime_comp_hours,
                     afternoon_hours, holiday_days, holiday_hours, vacation_days, sick_days, unpaid_days):
    exec_query("""
    INSERT INTO attendance(
        month_key, employee_id,
        hours_worked, overtime_hours, overtime_comp_hours,
        afternoon_hours, holiday_days, holiday_hours,
        vacation_days, sick_days, unpaid_days
    ) VALUES(?,?,?,?,?,?,?,?,?,?,?)
    ON CONFLICT(month_key, employee_id) DO UPDATE SET
        hours_worked=excluded.hours_worked,
        overtime_hours=excluded.overtime_hours,
        overtime_comp_hours=excluded.overtime_comp_hours,
        afternoon_hours=excluded.afternoon_hours,
        holiday_days=excluded.holiday_days,
        holiday_hours=excluded.holiday_hours,
        vacation_days=excluded.vacation_days,
        sick_days=excluded.sick_days,
        unpaid_days=excluded.unpaid_days
    """, (
        month_key, int(employee_id),
        float(hours_worked), float(overtime_hours), float(overtime_comp_hours),
        float(afternoon_hours), float(holiday_days), float(holiday_hours),
        float(vacation_days), float(sick_days), float(unpaid_days)
    ))

def upsert_extra_bonus(month_key, employee_id, pct):
    exec_query("""
    INSERT INTO extra_bonus(month_key, employee_id, pct)
    VALUES(?,?,?)
    ON CONFLICT(month_key, employee_id) DO UPDATE SET pct=excluded.pct
    """, (month_key, employee_id, float(pct)))

def upsert_eval(month_key, employee_id, area_code, score):
    exec_query("""
    INSERT INTO evaluations(month_key, employee_id, area_code, score)
    VALUES(?,?,?,?)
    ON CONFLICT(month_key, employee_id, area_code) DO UPDATE SET score=excluded.score
    """, (month_key, employee_id, area_code, float(score)))

def get_eval_score(month_key, employee_id, area_code):
    df = df_query("SELECT score FROM evaluations WHERE month_key=? AND employee_id=? AND area_code=?",
                  (month_key, employee_id, area_code))
    return float(df.iloc[0]["score"]) if not df.empty else 1.0

# ---------------- Business logic ----------------
def calc_bonus_fund(actual, plan, span1, span2, r1, r2, r3):
    diff = max(0.0, actual - plan)
    b = 0.0
    t1 = min(diff, span1); b += t1 * r1; diff -= t1
    t2 = min(diff, span2); b += t2 * r2; diff -= t2
    if diff > 0: b += diff * r3
    return b

@dataclass
class MonthParams:
    plan: float
    actual: float
    normative_hours: float
    overtime_limit_pct: float
    monthly_fund_hours: float
    min_attendance_pct: float
    tier1_span: float
    tier2_span: float
    r1: float
    r2: float
    r3: float
    afternoon_pct: float
    holiday_pct: float
    overtime_pct: float
    hours_per_day: float

def role_group(role: str) -> str:
    r = (role or "").lower()
    if "nákup" in r or "nakup" in r:
        return "NAKUP"
    if "technolog" in r:
        return "TECHNOLOG"
    if "kontrola kvality" in r:
        return "KVALITA"
    if "vedoucí výroby" in r:
        return "VEDOUCI"
    if "ceo" in r or "thp" in r:
        return "THP"
    return "VYROBA"

VAR_WEIGHTS = {
    # A) Operátoři ve výrobě (15/20/5 = 40 %)
    "VYROBA": {"DISC": 0.15, "QUAL": 0.20, "ASSET": 0.05},

    # B) Kontrola kvality (20/15/5 = 40 %)
    "KVALITA": {"QCTRL": 0.20, "NCR": 0.15, "ADMIN": 0.05},

    # D) Vedoucí výroby (25/10/5 = 40 %)
    "VEDOUCI": {"TERM": 0.25, "ORG": 0.10, "BOZP": 0.05},

    # E) Nákupčí materiálu (20/10/10 = 40 %)
    "NAKUP": {"NAK_TERM": 0.20, "NAK_SPEC": 0.10, "NAK_DOC": 0.10},

    # F) Technolog (20/5/15 = 40 %)
    "TECHNOLOG": {"TECH_OPT": 0.20, "TECH_FUNC": 0.05, "TECH_COMM": 0.15},
}

LABEL_MAP = {
    "DISC": "Dodržování pracovního řádu",
    "QUAL": "Kvalita práce / zmetky",
    "ASSET": "Péče o pracoviště a majetek",

    "QCTRL": "Správnost a důslednost kontroly",
    "NCR": "Prevence neshod a reklamací",
    "ADMIN": "Administrativa a spolupráce",

    "TERM": "Plnění termínů zakázek",
    "ORG": "Organizace výroby a řízení team leaderů",
    "BOZP": "Disciplína, BOZP a pořádek",

    "NAK_TERM": "Dodržení termínů dodávek",
    "NAK_SPEC": "Správnost nákupu (specifikace/množství/dokumentace)",
    "NAK_DOC": "Včasné dodání dokumentace směrem k zákazníkovi",

    "TECH_OPT": "Optimalizace CNC programů a technologie výroby",
    "TECH_FUNC": "Funkčnost technologie ve výrobě",
    "TECH_COMM": "Komunikace s výrobou a kontrolou kvality",
}

def money(x):
    return f"{x:,.0f} Kč".replace(",", " ")


def export_ucetni_odprac_xlsx(month_key: str, df_podklad: pd.DataFrame) -> bytes:
    """XLSX export podkladu pro účetní: pouze odpracované hodiny + bonusy/fix/extra."""
    bio = BytesIO()
    with pd.ExcelWriter(bio, engine="openpyxl") as writer:
        df_podklad.to_excel(writer, index=False, sheet_name="PODKLAD_ODPRAC")
        ws = writer.sheets["PODKLAD_ODPRAC"]
        ws.freeze_panes = "A2"
        for col_cells in ws.columns:
            max_len = 0
            col_letter = col_cells[0].column_letter
            for c in col_cells:
                if c.value is None:
                    continue
                max_len = max(max_len, len(str(c.value)))
            ws.column_dimensions[col_letter].width = min(max(10, max_len + 2), 45)
    return bio.getvalue()



def export_ucetni_podklad_xlsx(df_odprac: pd.DataFrame, df_nahrady: pd.DataFrame) -> bytes:
    """Jedno XLSX sešit se dvěma listy: PODKLAD_ODPRAC a PODKLAD_NAHRADY."""
    from openpyxl.styles import PatternFill
    bio = BytesIO()
    with pd.ExcelWriter(bio, engine="openpyxl") as writer:
        df_odprac.to_excel(writer, index=False, sheet_name="PODKLAD_ODPRAC")
        df_nahrady.to_excel(writer, index=False, sheet_name="PODKLAD_NAHRADY")

        yellow = PatternFill(start_color="FFFF00", end_color="FFFF00", fill_type="solid")

        # Highlight selected columns in PODKLAD_ODPRAC (5 sloupců)
        ws_o = writer.sheets["PODKLAD_ODPRAC"]
        headers = [c.value for c in ws_o[1]]
        highlight_cols = {
            "Odprac. hodiny",
            "Vyplacená Kč/h celkem",
            "Výkonnostní bonus (Kč)",
            "Extra bonus (Kč)",
            "Fix příplatek (Kč)",
        }
        for col_idx, h in enumerate(headers, start=1):
            if h in highlight_cols:
                for row in range(1, ws_o.max_row + 1):
                    ws_o.cell(row=row, column=col_idx).fill = yellow

        for name in ["PODKLAD_ODPRAC", "PODKLAD_NAHRADY"]:
            ws = writer.sheets[name]
            ws.freeze_panes = "A2"
            for col_cells in ws.columns:
                max_len = 0
                col_letter = col_cells[0].column_letter
                for c in col_cells:
                    if c.value is None:
                        continue
                    max_len = max(max_len, len(str(c.value)))
                ws.column_dimensions[col_letter].width = min(max(10, max_len + 2), 45)
    return bio.getvalue()




def export_podklad_mzdy_xlsx(month_key: str, month_fund_hours: float, mesic_info: dict, df_podklad: pd.DataFrame) -> bytes:
    """Create XLSX with sheets MESIC and PODKLAD_MZDY for accounting."""
    bio = BytesIO()
    with pd.ExcelWriter(bio, engine="openpyxl") as writer:
        mesic_rows = [
            {"Položka": "Měsíc", "Hodnota": month_key},
            {"Položka": "Fond hodin (měsíc)", "Hodnota": month_fund_hours},
        ]
        for k, v in mesic_info.items():
            mesic_rows.append({"Položka": k, "Hodnota": v})
        pd.DataFrame(mesic_rows).to_excel(writer, index=False, sheet_name="MESIC")
        df_podklad.to_excel(writer, index=False, sheet_name="PODKLAD_MZDY")

        for name in ["MESIC", "PODKLAD_MZDY"]:
            ws = writer.sheets[name]
            ws.freeze_panes = "A2"
            for col_cells in ws.columns:
                max_len = 0
                col_letter = col_cells[0].column_letter
                for c in col_cells:
                    if c.value is None:
                        continue
                    max_len = max(max_len, len(str(c.value)))
                ws.column_dimensions[col_letter].width = min(max(10, max_len + 2), 45)
    return bio.getvalue()



# ---------------- UI ----------------

def add_month_like(new_month_key: str, source_month_key: str):
    # Copy months row settings; initialize actual_revenue=0 (or keep? we'll set 0 by default)
    src = df_query("SELECT * FROM months WHERE month_key=?", (source_month_key,))
    if src.empty:
        raise ValueError("Zdrojový měsíc neexistuje.")
    row = src.iloc[0].to_dict()
    # fields to copy (keep plan & rates etc.)
    cols = table_columns("months")
    # ensure required columns
    row["month_key"] = new_month_key
    row["actual_revenue"] = 0.0
    # build insert dynamically
    insert_cols = [c for c in row.keys() if c in cols and c != "id"]
    placeholders = ",".join(["?"] * len(insert_cols))
    sql = f"INSERT INTO months({','.join(insert_cols)}) VALUES({placeholders})"
    exec_query(sql, tuple(row[c] for c in insert_cols))

def add_employee(full_name: str, role: str, coeff: float, counts_to_normative: int, receives_prod_bonus: int,
                 hourly_rate: float, fixed_allowance: float, allow_extra_bonus: int, is_active: int = 1):
    exec_query("""
    INSERT INTO employees(full_name, role, coeff, counts_to_normative, receives_prod_bonus,
                          hourly_rate, fixed_allowance, allow_extra_bonus, is_active)
    VALUES(?,?,?,?,?,?,?,?,?)
    """, (full_name, role, float(coeff), int(counts_to_normative), int(receives_prod_bonus),
          float(hourly_rate), float(fixed_allowance), int(allow_extra_bonus), int(is_active)))

st.title("Výpočet mezd a prémií (AKENG)")

months = df_query("SELECT month_key FROM months ORDER BY month_key DESC")["month_key"].tolist()
if not months:
    st.error("V databázi není žádný měsíc v tabulce months.")
    st.stop()

month_key = st.selectbox("Měsíc", months, index=0)
m = df_query("SELECT * FROM months WHERE month_key=?", (month_key,)).iloc[0]

tabs = st.tabs(["1) Nastavení", "2) Zaměstnanci", "3) Docházka", "4) Hodnocení", "5) Extra bonus", "6) Výsledky"])

with tabs[0]:
    st.subheader("Parametry měsíce")
    c1, c2, c3 = st.columns(3)
    plan = c1.number_input("Plán fakturace (Kč)", value=float(m["plan_revenue"]), step=10000.0)
    actual = c1.number_input("Skutečná fakturace (Kč)", value=float(m["actual_revenue"]), step=10000.0)
    normative_hours = c2.number_input("Normativ hodin (CNC)", value=float(m["normative_hours"]), step=1.0)
    overtime_limit_pct = c2.number_input("Limit přesčasů (% z normativu) – bonus gate", value=float(m["overtime_limit_pct"]), step=0.01, format="%.2f")
    monthly_fund_hours = c3.number_input("Měsíční fond hodin (např. 160)", value=float(m["monthly_fund_hours"]), step=1.0)
    min_attendance_pct = c3.number_input("Min. docházka pro prémii", value=float(m["min_attendance_pct"]), step=0.05, format="%.2f")

    st.markdown("**Bonusová pásma**")
    d1, d2, d3 = st.columns(3)
    tier1_span = d1.number_input("Pásmo 1 rozsah (Kč)", value=float(m["tier1_span"]), step=10000.0)
    r1 = d1.number_input("Pásmo 1 sazba", value=float(m["tier1_rate"]), step=0.01, format="%.2f")
    tier2_span = d2.number_input("Pásmo 2 rozsah (Kč)", value=float(m["tier2_span"]), step=10000.0)
    r2 = d2.number_input("Pásmo 2 sazba", value=float(m["tier2_rate"]), step=0.01, format="%.2f")
    r3 = d3.number_input("Pásmo 3 sazba", value=float(m["tier3_rate"]), step=0.01, format="%.2f")

    st.markdown("**Příplatky (výpočet mzdy)**")
    p1, p2, p3, p4 = st.columns(4)
    afternoon_pct = p1.number_input("Odpolední příplatek (0.10=10%)", value=float(m["afternoon_pct"]), step=0.01, format="%.2f")
    holiday_pct = p2.number_input("Svátek příplatek (1.00=100%)", value=float(m["holiday_pct"]), step=0.05, format="%.2f")
    overtime_pct = p3.number_input("Přesčas příplatek (0.25=25%)", value=float(m["overtime_pct"]), step=0.01, format="%.2f")
    hours_per_day = p4.number_input("Hodin/den (dovolená)", value=float(m["hours_per_day"]), step=0.5, format="%.1f")
    holiday_days = p4.number_input("Svatky (pracovni dny) v mesici", value=float(m["holiday_days"]), step=1.0, format="%.0f")
    # Mesicni fond (orientacne): pracovni dny Po-Pa minus svatky
    try:
        y, mo = map(int, month_key.split("-"))
        workdays = sum(1 for d in range(1, calendar.monthrange(y, mo)[1]+1)
                       if dt.date(y, mo, d).weekday() < 5)
        fund_days = max(0, workdays - int(holiday_days))
        fund_hours = fund_days * float(hours_per_day)
    except Exception:
        fund_days, fund_hours = 0, 0
    st.caption(f"Mesicni pracovni fond (orientacne): {fund_days} dnu / {fund_hours:.1f} hodin")


    if st.button("Uložit parametry"):
        exec_query("""
        UPDATE months SET plan_revenue=?, actual_revenue=?, normative_hours=?, overtime_limit_pct=?,
                          monthly_fund_hours=?, min_attendance_pct=?,
                          tier1_span=?, tier2_span=?, tier1_rate=?, tier2_rate=?, tier3_rate=?,
                          afternoon_pct=?, holiday_pct=?, overtime_pct=?, hours_per_day=?, holiday_days=?
        WHERE month_key=?
        """, (plan, actual, normative_hours, overtime_limit_pct, monthly_fund_hours, min_attendance_pct,
              tier1_span, tier2_span, r1, r2, r3, afternoon_pct, holiday_pct, overtime_pct, hours_per_day, holiday_days, month_key))
        st.success("Uloženo.")
    st.divider()
    st.subheader("Přidat nový měsíc")
    st.caption("Vytvoří nový měsíc tak, že zkopíruje nastavení z aktuálního měsíce. Docházka, hodnocení a extra bonusy budou pro nový měsíc prázdné (0).")
    new_month_key = st.text_input("Nový měsíc (YYYY-MM)", value="")
    if st.button("Vytvořit nový měsíc"):
        try:
            new_month_key = new_month_key.strip()
            if not new_month_key or len(new_month_key) != 7 or new_month_key[4] != "-":
                st.error("Zadej měsíc ve formátu YYYY-MM (např. 2026-03).")
            else:
                # check exists
                exists = df_query("SELECT 1 as x FROM months WHERE month_key=? LIMIT 1", (new_month_key,))
                if not exists.empty:
                    st.error("Tento měsíc už v databázi existuje.")
                else:
                    add_month_like(new_month_key, month_key)
                    st.success(f"Měsíc {new_month_key} vytvořen. Vyber ho nahoře v seznamu.")
        except Exception as e:
            st.error(f"Chyba při vytváření měsíce: {e}")

    st.divider()
    st.subheader("Smazat měsíc")
    st.warning("Pozor: Smazání měsíce odstraní i docházku, hodnocení a extra bonusy pro daný měsíc.")
    del_month_key = st.text_input("Měsíc ke smazání (YYYY-MM)", value="", key="del_month_key")
    confirm_del_month = st.checkbox("Potvrzuji, že chci měsíc nenávratně smazat", value=False, key="confirm_del_month")
    if st.button("Smazat měsíc", key="btn_del_month"):
        try:
            k = del_month_key.strip()
            if not confirm_del_month:
                st.error("Nejprve zaškrtni potvrzení smazání.")
            elif not k or len(k) != 7 or k[4] != "-":
                st.error("Zadej měsíc ve formátu YYYY-MM (např. 2026-03).")
            elif k == month_key:
                st.error("Nejprve přepni na jiný měsíc. Nelze smazat právě vybraný měsíc.")
            else:
                # delete dependent data
                exec_query("DELETE FROM attendance WHERE month_key=?", (k,))
                exec_query("DELETE FROM evaluations WHERE month_key=?", (k,))
                exec_query("DELETE FROM extra_bonus WHERE month_key=?", (k,))
                exec_query("DELETE FROM months WHERE month_key=?", (k,))
                st.success(f"Měsíc {k} byl smazán.")
        except Exception as e:
            st.error(f"Chyba při mazání měsíce: {e}")


with tabs[1]:
    st.subheader("Zaměstnanci – hodinová mzda a extra bonus")
    emp = df_query("""
        SELECT id, full_name AS Jméno, role AS Pozice, coeff AS Koeficient,
               counts_to_normative AS Do_normativu, receives_prod_bonus AS Bere_bonus,
               hourly_rate AS Hodinovka_Kc_h,
               hourly_rate_avg AS Prumerna_hodinovka, fixed_allowance AS Fix_priplatek,
               allow_extra_bonus AS Extra_bonus_povolen
        FROM employees WHERE is_active=1
        ORDER BY role, full_name
    """)
    # Zobrazeni jmen ve tvaru 'Prijmeni Jmeno'
    if "Jméno" in emp.columns:
        emp["Jméno"] = emp["Jméno"].map(format_name_surname_first)
        emp = sort_df_by_surname(emp, name_col="Jméno")



    # Odvozené hodinové sazby dle MV (60/40)
    # Guard: ensure Prumerna_hodinovka column exists
    if "Prumerna_hodinovka" not in emp.columns:
        emp["Prumerna_hodinovka"] = emp["Hodinovka_Kc_h"]
    emp["Narokova_hodinovka_60"] = (emp["Hodinovka_Kc_h"] * 0.60).round(2)
    emp["Nenarokova_hodinovka_40_max"] = (emp["Hodinovka_Kc_h"] * 0.40).round(2)
    # Pokud není průměrná hodinovka vyplněná (0), použijeme běžnou hodinovku
    if "Prumerna_hodinovka" in emp.columns:
        emp.loc[emp["Prumerna_hodinovka"].fillna(0) <= 0, "Prumerna_hodinovka"] = emp.loc[emp["Prumerna_hodinovka"].fillna(0) <= 0, "Hodinovka_Kc_h"]
    else:
        emp["Prumerna_hodinovka"] = emp["Hodinovka_Kc_h"]

    edited = st.data_editor(emp, width="stretch", hide_index=True,
                            disabled=["id","Jméno","Pozice","Narokova_hodinovka_60","Nenarokova_hodinovka_40_max"])
    if st.button("Uložit hodinovky/extra bonus"):
        for _, r in edited.iterrows():
            exec_query("UPDATE employees SET hourly_rate=?, allow_extra_bonus=? WHERE id=?",
                       (float(r["Hodinovka_Kc_h"]), int(r["Extra_bonus_povolen"]), int(r["id"])))
        st.success("Uloženo.")
    st.divider()
    
    st.divider()
    st.subheader("Smazat / deaktivovat zaměstnance")
    st.caption("Doporučeno: zaměstnance nemažte, ale deaktivujte (neztratí se historie).")
    emp_all = df_query("SELECT id, full_name || ' — ' || role AS label, is_active FROM employees ORDER BY is_active DESC, role, full_name")
    if not emp_all.empty:
        emp_map = dict(zip(emp_all["label"], emp_all["id"]))
        selected_label = st.selectbox("Vyber zaměstnance", list(emp_map.keys()), key="sel_del_emp")
        emp_id = int(emp_map[selected_label])
        col_a, col_b = st.columns(2)
        with col_a:
            if st.button("Deaktivovat (doporučeno)", key="btn_deactivate_emp"):
                try:
                    exec_query("UPDATE employees SET is_active=0 WHERE id=?", (emp_id,))
                    st.success("Zaměstnanec byl deaktivován.")
                except Exception as e:
                    st.error(f"Chyba: {e}")
        with col_b:
            confirm_hard = st.checkbox("Chci zaměstnance nenávratně smazat (včetně historie)", value=False, key="confirm_hard_emp")
            if st.button("Smazat (nevratné)", key="btn_delete_emp"):
                if not confirm_hard:
                    st.error("Nejprve zaškrtni potvrzení nevratného smazání.")
                else:
                    try:
                        # delete dependent history rows
                        exec_query("DELETE FROM attendance WHERE employee_id=?", (emp_id,))
                        exec_query("DELETE FROM evaluations WHERE employee_id=?", (emp_id,))
                        exec_query("DELETE FROM extra_bonus WHERE employee_id=?", (emp_id,))
                        exec_query("DELETE FROM employees WHERE id=?", (emp_id,))
                        st.success("Zaměstnanec byl smazán.")
                    except Exception as e:
                        st.error(f"Chyba: {e}")
    else:
        st.info("V databázi nejsou žádní zaměstnanci.")

    st.subheader("Přidat nového zaměstnance")
    st.caption("Přidá zaměstnance do seznamu. Nastav role, koeficient, zda se počítá do normativu a zda bere výkonnostní bonus.")
    with st.form("add_employee_form"):
        col1, col2 = st.columns(2)
        full_name = col1.text_input("Jméno a příjmení", value="")
        role = col2.text_input("Pozice / role", value="")
        col3, col4, col5 = st.columns(3)
        coeff = col3.number_input("Koeficient", value=1.0, step=0.1, format="%.2f")
        counts_to_normative = col4.selectbox("Počítá se do normativu (CNC)?", options=[0,1], index=0)
        receives_prod_bonus = col5.selectbox("Bere výkonnostní bonus?", options=[0,1], index=0)
        col6, col7, col8 = st.columns(3)
        hourly_rate = col6.number_input("Hodinová mzda (Kč/h)", value=0.0, step=10.0)
        fixed_allowance = col7.number_input("Fixní příplatek (Kč/měsíc)", value=0.0, step=500.0)
        allow_extra_bonus = col8.selectbox("Povolit extra bonus (0/5/10/20/30 %)?", options=[0,1], index=0)
        is_active = st.checkbox("Aktivní", value=True)
        submitted = st.form_submit_button("Přidat zaměstnance")
        if submitted:
            if not full_name.strip() or not role.strip():
                st.error("Vyplň jméno i pozici.")
            else:
                try:
                    add_employee(full_name.strip(), role.strip(), coeff, counts_to_normative, receives_prod_bonus,
                                 hourly_rate, fixed_allowance, allow_extra_bonus, 1 if is_active else 0)
                    st.success("Zaměstnanec přidán. Obnov stránku nebo přepni záložku pro zobrazení.")
                except Exception as e:
                    st.error(f"Chyba při přidání zaměstnance: {e}")
    
    
with tabs[2]:
    st.subheader("Docházka + příplatky + absence")
    st.caption("Odpracované hodiny = čistě odpracované hodiny (bez přesčasů a bez svátků). Svátky vyplň do sloupců 'Svatky' (dny) a 'Svatky_hodiny'. Přesčasy zadávej zvlášť. Náhradní volno = počet přesčasových hodin, za které se nevyplatí příplatek.")
    att = df_query("""
        SELECT e.id, e.full_name AS Jméno, e.role AS Pozice,
               COALESCE(a.hours_worked,0) AS Odpracovane_hodiny,
               COALESCE(a.overtime_hours,0) AS Prescasy,
               COALESCE(a.overtime_comp_hours,0) AS Prescasy_nahradni_volno,
               COALESCE(a.afternoon_hours,0) AS Odpoledni_hodiny,
               COALESCE(a.holiday_days,0) AS Svatky,
          COALESCE(a.holiday_hours,0) AS Svatky_hodiny,
               COALESCE(a.vacation_days,0) AS Dovolena_dny,
               COALESCE(a.sick_days,0) AS PN_dny,
               COALESCE(a.unpaid_days,0) AS Neplacene_dny
        FROM employees e
        LEFT JOIN attendance a ON a.employee_id=e.id AND a.month_key=?
        WHERE e.is_active=1
        ORDER BY e.role, e.full_name
    """, (month_key,))
    if "Jméno" in att.columns:
        att["Jméno"] = att["Jméno"].map(format_name_surname_first)
        att = sort_df_by_surname(att, name_col="Jméno")

    edited = st.data_editor(att, width="stretch", hide_index=True, disabled=["id","Jméno","Pozice"])
    if st.button("Uložit docházku"):
        for _, r in edited.iterrows():
            upsert_attendance(month_key, int(r["id"]), r["Odpracovane_hodiny"], r["Prescasy"], r["Prescasy_nahradni_volno"],
                              r["Odpoledni_hodiny"], r["Svatky"], r["Svatky_hodiny"], r["Dovolena_dny"], r["PN_dny"], r["Neplacene_dny"])
        st.success("Docházka uložena.")

with tabs[3]:
    st.subheader("Hodnocení 100/50/0 (pohyblivá složka 40 %)")
    emp = df_query("SELECT id, full_name, role FROM employees WHERE is_active=1 ORDER BY role, full_name")
    rows = []
    for _, r in emp.iterrows():
        g = role_group(r["role"])
        if g not in VAR_WEIGHTS:
            continue
        for area in VAR_WEIGHTS[g].keys():
            rows.append({"id": int(r["id"]), "Jméno": r["full_name"], "Pozice": r["role"], "area_code": area, "Oblast": LABEL_MAP.get(area, area)})
    grid = pd.DataFrame(rows)
    existing = df_query("SELECT employee_id, area_code, score FROM evaluations WHERE month_key=?", (month_key,))
    grid = grid.merge(existing, how="left", left_on=["id","area_code"], right_on=["employee_id","area_code"])
    grid["score"] = grid["score"].fillna(1.0)
    edited = st.data_editor(grid[["id","Jméno","Pozice","Oblast","area_code","score"]].rename(columns={"score":"Hodnocení (1.0/0.5/0.0)"}),
                            width="stretch", hide_index=True, disabled=["id","Jméno","Pozice","Oblast","area_code"])
    if st.button("Uložit hodnocení"):
        for _, r in edited.iterrows():
            upsert_eval(month_key, int(r["id"]), str(r["area_code"]), float(r["Hodnocení (1.0/0.5/0.0)"]))
        st.success("Uloženo.")

with tabs[4]:
    st.subheader("Extra bonus 0/5/10/20/30 % (kontrola / vedoucí / THP / CEO)")
    eligible = df_query("SELECT id, full_name AS Jméno, role AS Pozice FROM employees WHERE is_active=1 AND allow_extra_bonus=1 ORDER BY role, full_name")
    if eligible.empty:
        st.info("Nikdo nemá povolený extra bonus (nastavíš v záložce Zaměstnanci).")
    else:
        existing = df_query("SELECT employee_id, pct FROM extra_bonus WHERE month_key=?", (month_key,))
        eligible = eligible.merge(existing, how="left", left_on="id", right_on="employee_id")
        eligible["pct"] = eligible["pct"].fillna(0.0)
        options = [0, 0.05, 0.10, 0.20, 0.30]
        def fmt(x): return f"{int(x*100)} %"
        chosen = []
        for _, r in eligible.iterrows():
            cols = st.columns([3,1])
            cols[0].write(f"{r['Jméno']} — {r['Pozice']}")
            pct = cols[1].selectbox(" ", options, index=options.index(float(r["pct"])) if float(r["pct"]) in options else 0,
                                    key=f"extra_{int(r['id'])}", format_func=fmt)
            chosen.append((int(r["id"]), float(pct)))
        if st.button("Uložit extra bonusy"):
            for emp_id, pct in chosen:
                upsert_extra_bonus(month_key, emp_id, pct)
            st.success("Uloženo.")

with tabs[5]:
    st.subheader("Výsledky – hrubá mzda (hodinovka + příplatky + 60/40 + bonusy)")

    m = df_query("SELECT * FROM months WHERE month_key=?", (month_key,)).iloc[0]
    mp = MonthParams(
        plan=float(m["plan_revenue"]),
        actual=float(m["actual_revenue"]),
        normative_hours=float(m["normative_hours"]),
        overtime_limit_pct=float(m["overtime_limit_pct"]),
        monthly_fund_hours=float(m["monthly_fund_hours"]),
        min_attendance_pct=float(m["min_attendance_pct"]),
        tier1_span=float(m["tier1_span"]),
        tier2_span=float(m["tier2_span"]),
        r1=float(m["tier1_rate"]),
        r2=float(m["tier2_rate"]),
        r3=float(m["tier3_rate"]),
        afternoon_pct=float(m["afternoon_pct"]),
        holiday_pct=float(m["holiday_pct"]),
        overtime_pct=float(m["overtime_pct"]),
        hours_per_day=float(m["hours_per_day"]),
    )
    # Standardni delka smeny (hodiny/den) pro prepocet svatku (pokud neni nastavena, bereme 8h)
    day_hours_f = float(getattr(mp, "hours_per_day", 8.0)) if float(getattr(mp, "hours_per_day", 0.0) or 0.0) > 0 else 8.0
    # Fallback sazby (pokud by byly v Nastavení nulové)
    overtime_pct_f = float(mp.overtime_pct) if float(mp.overtime_pct) > 0 else 0.25
    afternoon_pct_f = float(mp.afternoon_pct) if float(mp.afternoon_pct) > 0 else 0.10
    holiday_pct_f = float(mp.holiday_pct) if float(mp.holiday_pct) > 0 else 1.00

    data = df_query("""
        SELECT e.id, e.full_name, e.role, e.coeff, e.counts_to_normative, e.receives_prod_bonus,
               e.hourly_rate, e.hourly_rate_avg, e.fixed_allowance, e.allow_extra_bonus,
               COALESCE(a.hours_worked,0) AS hours_worked,
               COALESCE(a.overtime_hours,0) AS overtime_hours,
               COALESCE(a.overtime_comp_hours,0) AS overtime_comp_hours,
               COALESCE(a.afternoon_hours,0) AS afternoon_hours,
               COALESCE(a.holiday_days,0) AS holiday_days,
               COALESCE(a.holiday_hours,0) AS holiday_hours,
               COALESCE(a.vacation_days,0) AS vacation_days,
               COALESCE(a.sick_days,0) AS sick_days,
               COALESCE(a.unpaid_days,0) AS unpaid_days,
               COALESCE(e.hourly_rate_avg,0) AS avg_hourly_rate
        FROM employees e
        LEFT JOIN attendance a ON a.employee_id=e.id AND a.month_key=?
        WHERE e.is_active=1
        ORDER BY e.role, e.full_name
    """, (month_key,))

    # Defensive: drop duplicate columns (can happen after merges / duplicate names)

    data = data.loc[:, ~data.columns.duplicated()].copy()
    # Průměrná hodinovka pro náhrady se bere ze záložky Zaměstnanci (hourly_rate_avg)
    if "hourly_rate_avg" in data.columns:
        data["avg_hourly_rate"] = pd.to_numeric(data["hourly_rate_avg"], errors="coerce").fillna(0.0)

    # --- MAPOVANI SVATECNICH HODIN ---
    # Dochazka pouziva Svatky_hodiny, vysledky pouzivaji holiday_hours
    if "holiday_hours" not in data.columns and "Svatky_hodiny" in data.columns:
        data["holiday_hours"] = pd.to_numeric(data["Svatky_hodiny"], errors="coerce").fillna(0.0)
    elif "holiday_hours" in data.columns:
        data["holiday_hours"] = pd.to_numeric(data["holiday_hours"], errors="coerce").fillna(0.0)


    # Ensure avg_hourly_rate and hourly_rate are numeric Series

    if 'avg_hourly_rate' in data.columns:

        data['avg_hourly_rate'] = pd.to_numeric(data['avg_hourly_rate'], errors='coerce').fillna(0.0)

    if 'hourly_rate' in data.columns:

        data['hourly_rate'] = pd.to_numeric(data['hourly_rate'], errors='coerce').fillna(0.0)

    # Defensive: if duplicate columns exist after SQL/merge, keep first occurrence

    if isinstance(data.get("avg_hourly_rate", None), pd.DataFrame):

        data["avg_hourly_rate"] = data["avg_hourly_rate"].iloc[:, 0]

    if isinstance(data.get("hourly_rate", None), pd.DataFrame):

        data["hourly_rate"] = data["hourly_rate"].iloc[:, 0]

    # Bonus gate: CNC OT only
    cnc = data[data["counts_to_normative"]==1].copy()
    total_ot_cnc = float(cnc["overtime_hours"].sum())
    ot_limit = mp.normative_hours * mp.overtime_limit_pct
    bonus_fund = calc_bonus_fund(mp.actual, mp.plan, mp.tier1_span, mp.tier2_span, mp.r1, mp.r2, mp.r3)
    bonus_gate = (mp.actual > mp.plan) and (total_ot_cnc <= ot_limit + 1e-9)

    # Bonus distribution
    min_hours_for_bonus = mp.monthly_fund_hours * mp.min_attendance_pct
    rec = data[(data["receives_prod_bonus"]==1) & (data["hours_worked"]>=min_hours_for_bonus)].copy()
    rec["w"] = rec["hours_worked"] * rec["coeff"]
    total_w = float(rec["w"].sum())
    rec["prod_bonus"] = 0.0
    if bonus_gate and bonus_fund > 0 and total_w > 0:
        rec["prod_bonus"] = bonus_fund * (rec["w"] / total_w)

    data = data.merge(rec[["id","prod_bonus"]], on="id", how="left")
    data["prod_bonus"] = data["prod_bonus"].fillna(0.0)

    # Paid hours: regular + overtime + vacation
    # Robust monthly average hourly rate (entered in Docházka as Prumerna_hodinovka)
    avg_col = data.loc[:, "avg_hourly_rate"]
    if isinstance(avg_col, pd.DataFrame):
        avg_col = avg_col.iloc[:, 0]
    hr_col = data.loc[:, "hourly_rate"]
    if isinstance(hr_col, pd.DataFrame):
        hr_col = hr_col.iloc[:, 0]
    avg_col = pd.to_numeric(avg_col, errors="coerce").fillna(0.0)
    hr_col = pd.to_numeric(hr_col, errors="coerce").fillna(0.0)
    data["avg_rate_used"] = np.where(pd.to_numeric(data.get("avg_hourly_rate", 0.0), errors="coerce").fillna(0.0) > 0,
                                   pd.to_numeric(data.get("avg_hourly_rate", 0.0), errors="coerce").fillna(0.0),
                                   pd.to_numeric(data.get("hourly_rate", 0.0), errors="coerce").fillna(0.0))
    data["vacation_hours"] = data["vacation_days"] * mp.hours_per_day
    data["worked_hours_total"] = data["hours_worked"] + data["overtime_hours"]
    data["paid_hours_total"] = data["worked_hours_total"] + data["vacation_hours"]

    # Average hourly wage for replacements (vacation/holiday premium/overtime premium)
    # Compute variable fraction from evaluations (max 0.40)
    def var_frac(row):
        g = role_group(row["role"])
        if g not in VAR_WEIGHTS:
            return 0.0
        frac = 0.0
        for area, w in VAR_WEIGHTS[g].items():
            frac += w * get_eval_score(month_key, int(row["id"]), area)
        return float(frac)

    data["var_frac"] = data.apply(var_frac, axis=1)

    # Hourly split per MV: 60% nároková + (vyplacená část z max 40% dle hodnocení)
    data["hourly_fixed"] = data["hourly_rate"] * 0.60
    data["hourly_variable_paid"] = data["hourly_rate"] * data["var_frac"]
    data["hourly_total_paid"] = data["hourly_fixed"] + data["hourly_variable_paid"]

    # Wage for worked hours (including overtime hours at base rate)
    data["pay_fixed_worked"] = data["hourly_fixed"] * data["worked_hours_total"]
    data["pay_variable_worked"] = data["hourly_variable_paid"] * data["worked_hours_total"]

    # Vacation pay uses average hourly wage
    data["pay_vacation"] = data["avg_rate_used"] * data["vacation_hours"]

    # Base wage total (without premiums/bonuses)
    data["base_wage_total"] = data["pay_fixed_worked"] + data["pay_variable_worked"] + data["pay_vacation"]

    # Premiums (computed from average hourly wage)
    data["premium_afternoon"] = data["avg_hourly_rate"] * data["afternoon_hours"] * mp.afternoon_pct
    # Robust svatek (100% priplatek) – zakon: prumerna hodinovka * svatecni hodiny * holiday_pct
    if "holiday_hours" not in data.columns and "Svatky_hodiny" in data.columns:
        data["holiday_hours"] = data["Svatky_hodiny"]
    data["holiday_hours"] = pd.to_numeric(data.get("holiday_hours", 0.0), errors="coerce").fillna(0.0)
    data["premium_holiday"] = data["avg_rate_used"] * data.get("holiday_hours",0.0) * (float(mp.holiday_pct) if float(mp.holiday_pct)>0 else 1.0)

    # Overtime premium (25%) only for overtime hours not taken as compensatory time
    data["overtime_comp_hours"] = data["overtime_comp_hours"].clip(lower=0.0)
    # Odpracované hodiny bez svátku (svátek se vede zvlášť)
    data["hours_regular"] = (pd.to_numeric(data.get("hours_worked", 0.0), errors="coerce").fillna(0.0)
                           - pd.to_numeric(data.get("holiday_hours", 0.0), errors="coerce").fillna(0.0)).clip(lower=0.0)
    # Svatek: uzivatel zadava Svatky (dny) a Svatky_hodiny (hodiny prace ve svatek)
    data["holiday_days"] = pd.to_numeric(data.get("holiday_days", 0.0), errors="coerce").fillna(0.0)
    data["holiday_hours"] = pd.to_numeric(data.get("holiday_hours", 0.0), errors="coerce").fillna(0.0)
    # Hodiny svatku, kdy zamestnanec NEpracoval (nahrada 100% prumeru)
    data["holiday_comp_hours"] = (data["holiday_days"] * day_hours_f - data["holiday_hours"]).clip(lower=0.0)
    data["overtime_premium_hours"] = (data["overtime_hours"] - data["overtime_comp_hours"]).clip(lower=0.0)
    data["premium_overtime"] = data["avg_hourly_rate"] * data["overtime_premium_hours"] * mp.overtime_pct

    # For reporting: "Základ (100%)" is hourly_rate * paid_hours_total
    data["base_pay"] = data["hourly_rate"] * (data.get("hours_worked",0.0) + data.get("overtime_hours",0.0) + data.get("holiday_hours",0.0)) + data["avg_rate_used"] * data.get("vacation_hours",0.0) + data["avg_rate_used"] * data.get("holiday_comp_hours",0.0)
    data["fixed_part"] = data["pay_fixed_worked"]  # 60% nároková část za odpracované hodiny
    data["variable_component"] = data["pay_variable_worked"]  # vyplacená nenároková část za odpracované hodiny

    
    def var_component(row):
        g = role_group(row["role"])
        if g not in VAR_WEIGHTS:
            return 0.0
        frac = 0.0
        for area, w in VAR_WEIGHTS[g].items():
            frac += w * get_eval_score(month_key, int(row["id"]), area)
        return float(row["base_pay"]) * frac

    data["variable_component"] = data.apply(var_component, axis=1)

    # Gross before extra bonus

    # --- HRUBA MZDA (spravne 60/40 varianta A) ---
    # Zaklad (100%) = hodinovka*(odpracovane hodiny + prescasy) + prumerna_hodinovka*dovolena_hodiny
    # Narokova slozka = 60% ze zakladu
    # Pohybliva slozka = zaklad * (vahy * score)  (max 40%)
    # Priplatky a bonusy jsou NAD ramec (pricitaji se az potom)

    # Ensure fixed part is 60% of base
    data["fixed_part"] = (data["base_pay"] * 0.60)

    # Gross before extra = 60% + vyplacena pohybliva + priplatky + fix + vyrobni bonus
    data["gross_before_extra"] = (
        data["fixed_part"]
        + data["variable_component"]
        + data["premium_afternoon"]
        + data["premium_holiday"]
        + data["premium_overtime"]
        + data["fixed_allowance"]
        + data["prod_bonus"]
    )

    # Extra bonus is percent of gross_before_extra (if allowed)
    # Ensure pct column exists (extra bonus %); default 0
    if "pct" not in data.columns:
        data["pct"] = 0.0
    data["extra_bonus_amount"] = data["gross_before_extra"] * data["pct"]

    # Final gross total
    data["gross_total"] = data["gross_before_extra"] + data["extra_bonus_amount"]

    # Rounding
    for c in ["base_pay","fixed_part","variable_component","premium_afternoon","premium_holiday","premium_overtime","fixed_allowance","prod_bonus","extra_bonus_amount","gross_total"]:
        if c in data.columns:
            data[c] = data[c].round(2)
    data["gross_total_rounded"] = data["gross_total"].round(0)


    st.markdown(f"**Bonusový fond:** {money(bonus_fund)} | **Podmínky bonusu:** {'SPLNĚNY' if bonus_gate else 'NESPLNĚNY'} | CNC přesčasy {total_ot_cnc:.1f} h / limit {ot_limit:.1f} h")

    # --- FINAL RECOMPUTE of premiums to ensure UI matches export ---

    # Ensure numeric

    data["hours_worked"] = pd.to_numeric(data.get("hours_worked", 0.0), errors="coerce").fillna(0.0)

    data["overtime_hours"] = pd.to_numeric(data.get("overtime_hours", 0.0), errors="coerce").fillna(0.0)

    data["overtime_comp_hours"] = pd.to_numeric(data.get("overtime_comp_hours", 0.0), errors="coerce").fillna(0.0)

    data["afternoon_hours"] = pd.to_numeric(data.get("afternoon_hours", 0.0), errors="coerce").fillna(0.0)

    data["holiday_hours"] = pd.to_numeric(data.get("holiday_hours", 0.0), errors="coerce").fillna(0.0)


    # Fallback sazby

    overtime_pct_f = float(mp.overtime_pct) if float(mp.overtime_pct) > 0 else 0.25

    afternoon_pct_f = float(mp.afternoon_pct) if float(mp.afternoon_pct) > 0 else 0.10

    holiday_pct_f = float(mp.holiday_pct) if float(mp.holiday_pct) > 0 else 1.00


    # Přesčasové hodiny pro příplatek (po odečtu náhradního volna)

    data["overtime_premium_hours"] = (data["overtime_hours"] - data["overtime_comp_hours"]).clip(lower=0.0)


    # Průměr pro náhrady (avg_rate_used) – pokud chybí, fallback na hodinovku

    if "avg_rate_used" not in data.columns:

        data["avg_rate_used"] = pd.to_numeric(data.get("hourly_rate", 0.0), errors="coerce").fillna(0.0)

    else:

        data["avg_rate_used"] = pd.to_numeric(data["avg_rate_used"], errors="coerce").fillna(0.0)


    data["hourly_rate"] = pd.to_numeric(data.get("hourly_rate", 0.0), errors="coerce").fillna(0.0)


    # Recompute premiums

    data["premium_overtime"] = (data["avg_rate_used"] * data["overtime_premium_hours"] * overtime_pct_f).round(2)

    data["premium_afternoon"] = (data["hourly_rate"] * data["afternoon_hours"] * afternoon_pct_f).round(2)

    data["premium_holiday"] = (data["avg_rate_used"] * data["holiday_hours"] * holiday_pct_f).round(2)


    # Řazení výsledků podle příjmení a zobrazení 'Příjmení Jméno'


    data = data.copy()


    data["Jméno"] = data["full_name"].map(format_name_surname_first)


    data = data.assign(_sort_surname=data["full_name"].map(lambda s: split_name_first_surname(s)[1].lower()),


                       _sort_first=data["full_name"].map(lambda s: split_name_first_surname(s)[0].lower()))


    data["_sort_surname"] = data["_sort_surname"].where(data["_sort_surname"] != "", data["full_name"].str.lower())


    data = data.sort_values(by=["_sort_surname","_sort_first"]).drop(columns=["_sort_surname","_sort_first"]).reset_index(drop=True)


    # Force display name as 'Příjmení Jméno' in results


    if "Jméno" in data.columns:


        data["Jméno"] = data["Jméno"].map(format_name_surname_first)


        data = sort_df_by_surname(data, name_col="Jméno")


    elif "full_name" in data.columns:


        data["Jméno"] = data["full_name"].map(format_name_surname_first)


        data = sort_df_by_surname(data, name_col="Jméno")


    # FORCE: always create display name 'Prijmeni Jmeno' from full_name


    data = data.copy()


    if "full_name" in data.columns:


        data["Jmeno_display"] = data["full_name"].map(format_name_surname_first)


    else:


        data["Jmeno_display"] = data.get("Jméno", "").map(format_name_surname_first) if "Jméno" in data.columns else ""


    # Sort by surname using the display column


    tmp_sort = pd.DataFrame({"Jméno": data["Jmeno_display"]})


    tmp_sort = sort_df_by_surname(tmp_sort, name_col="Jméno")


    data = data.reindex(tmp_sort.index).reset_index(drop=True)


    view = data[[
        "Jmeno_display",
        "role",
        "hourly_rate",
        "hours_worked",
        "overtime_hours",
        "overtime_comp_hours",
        "overtime_premium_hours",
        "vacation_days",
        "sick_days",
        "unpaid_days",
        "paid_hours_total",
        "base_pay",
        "fixed_part",
        "variable_component",
        "premium_afternoon",
        "premium_holiday",
        "premium_overtime",
        "fixed_allowance",
        "prod_bonus",
        "pct",
        "extra_bonus_amount",
        "gross_total_rounded"
    ]].copy()

    view.columns = [
        "Jméno",
        "Pozice",
        "Hodinovka (Kč/h)",
        "Odpracovane_hodiny",
        "Přesčasy",
        "Přesčasy (náhradní volno)",
        "Přesčasové hodiny pro příplatek",
        "Dovolená (dny)",
        "PN (dny)",
        "Neplacené (dny)",
        "Placené hodiny",
        "Základ (100%)",
        "Nároková složka (60%)",
        "Pohyblivá složka (40%)",
        "Přípl. odpolední",
        "Přípl. svátek",
        "Přípl. přesčas (25%)",
        "Fix příplatek",
        "Výkonnostní bonus",
        "Extra %",
        "Extra Kč",
        "Hrubá mzda CELKEM (zaokrouhleno)"
    ]

    st.dataframe(view, width="stretch", hide_index=True)
    st.download_button("Stáhnout CSV", data=view.to_csv(index=False).encode("utf-8"), file_name=f"mzdy_{month_key}.csv", mime="text/csv")

    # Export pro účetní (1 soubor XLSX se 2 listy)
    # PODKLAD_ODPRAC: 60/40 hodinovky * odpracované hodiny + bonusy/fix/extra
    # PODKLAD_NAHRADY: dovolená, svátky (dny + hodiny práce), přesčasy, odpolední, PN/Neplacené (evidence)
    try:
        # koeficient vyplacení nenárokové složky (0..0.40) dle hodnocení 100/50/0
        def _score_frac(row):
            g = role_group(row["role"])
            if g not in VAR_WEIGHTS:
                return 0.0
            frac = 0.0
            for area_code, w in VAR_WEIGHTS[g].items():
                frac += w * get_eval_score(month_key, int(row["id"]), area_code)
            return float(frac)

        data["var_frac"] = data.apply(_score_frac, axis=1)  # 0..0.40

        hours_work = pd.to_numeric(data.get("hours_worked", 0.0), errors="coerce").fillna(0.0)
        h100 = pd.to_numeric(data.get("hourly_rate", 0.0), errors="coerce").fillna(0.0)

        h60 = (h100 * 0.60).round(2)
        h40_max = (h100 * 0.40).round(2)
        h40_paid = (h100 * data["var_frac"]).round(2)
        h_paid = (h60 + h40_paid).round(2)

        mzda_odprac = (h_paid * hours_work).round(2)

        fix = pd.to_numeric(data.get("fixed_allowance", 0.0), errors="coerce").fillna(0.0).round(2)
        vyk_bonus = pd.to_numeric(data.get("prod_bonus", 0.0), errors="coerce").fillna(0.0).round(2)
        extra = pd.to_numeric(data.get("extra_bonus_amount", 0.0), errors="coerce").fillna(0.0).round(2)

        hruba_bez_nahrad = (mzda_odprac + fix + vyk_bonus + extra).round(0)

        df_odprac = pd.DataFrame({
            "Jméno": data["Jméno"] if "Jméno" in data.columns else data["full_name"],
            "Pozice": data["role"],
            "Odprac. hodiny": hours_work.round(2),
            "Hodinovka 100%": h100.round(2),
            "Nároková Kč/h (60%)": h60,
            "Nenároková max Kč/h (40%)": h40_max,
            "Nenároková vyplac. Kč/h": h40_paid,
            "Vyplacená Kč/h celkem": h_paid,
                        "Výkonnostní bonus (Kč)": vyk_bonus,
            "Extra bonus (Kč)": extra,
            "Fix příplatek (Kč)": fix,
                    })
        df_odprac["Jméno"] = df_odprac["Jméno"].map(format_name_surname_first)
        df_odprac = sort_df_by_surname(df_odprac, name_col="Jméno")

        # Náhrady / příplatky dle hodin (účetní dopočítává svátek doma jako 8h z průměru)
        vac_hours = pd.to_numeric(data.get("vacation_hours", 0.0), errors="coerce").fillna(0.0)
        hol_days = pd.to_numeric(data.get("holiday_days", 0.0), errors="coerce").fillna(0.0)  # Svátky (dny)
        hol_work_hours = pd.to_numeric(data.get("holiday_hours", 0.0), errors="coerce").fillna(0.0)  # Hodiny ve svátek v práci
        ot_hours = pd.to_numeric(data.get("overtime_hours", 0.0), errors="coerce").fillna(0.0)
        ot_nv = pd.to_numeric(data.get("overtime_comp_hours", 0.0), errors="coerce").fillna(0.0)
        ot_prem_h = (ot_hours - ot_nv).clip(lower=0.0)
        aft_h = pd.to_numeric(data.get("afternoon_hours", 0.0), errors="coerce").fillna(0.0)
        pn_days = pd.to_numeric(data.get("sick_days", 0.0), errors="coerce").fillna(0.0)
        unpaid_days = pd.to_numeric(data.get("unpaid_days", 0.0), errors="coerce").fillna(0.0)

        avg_rate = pd.to_numeric(data.get("avg_rate_used", h100), errors="coerce").fillna(0.0)

        # Příplatky (informativně)
        overtime_pct_f = float(mp.overtime_pct) if float(mp.overtime_pct) > 0 else 0.25
        afternoon_pct_f = float(mp.afternoon_pct) if float(mp.afternoon_pct) > 0 else 0.10
        pay_ot_prem = (avg_rate * ot_prem_h * overtime_pct_f).round(2)
        pay_aft = (h100 * aft_h * afternoon_pct_f).round(2)

        df_nahrady = pd.DataFrame({
            "Jméno": data["Jméno"] if "Jméno" in data.columns else data["full_name"],
            "Pozice": data["role"],
            "Průměr náhrady (Kč/h)": avg_rate.round(2),
            "Dovolená hodiny": vac_hours.round(2),
            "Svátky (dny)": hol_days.round(2),
            "Hodiny ve svátek (práce)": hol_work_hours.round(2),
            "Přesčas hodiny": ot_hours.round(2),
            "Přesčas NV hodiny": ot_nv.round(2),
            "Přesčas pro příplatek (h)": ot_prem_h.round(2),
            "Příplatek přesčas 25% (Kč)": pay_ot_prem,
            "Odpolední hodiny": aft_h.round(2),
            "Příplatek odpolední 5% (Kč)": pay_aft,
            "Doba nemoci (evidence)": pn_days.round(2),
            "Neplacené dny (evidence)": unpaid_days.round(2),
        })
        df_nahrady["Jméno"] = df_nahrady["Jméno"].map(format_name_surname_first)
        df_nahrady = sort_df_by_surname(df_nahrady, name_col="Jméno")

        # Final sort for export (match Výsledky order)
        df_odprac = sort_df_by_surname(df_odprac, name_col="Jméno")
        df_nahrady = sort_df_by_surname(df_nahrady, name_col="Jméno")
        xlsx_bytes = export_ucetni_podklad_xlsx(df_odprac, df_nahrady)
        st.download_button(
            "Export podklad pro účetní (XLSX)",
            data=xlsx_bytes,
            file_name=f"podklad_ucetni_{month_key}.xlsx",
            mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        )
    except Exception as e:
        st.warning(f"Export pro účetní se nepodařil: {e}")

    st.caption("Model 60/40: hrubý základ (hodinovka × placené hodiny) se dělí na 60% nárokovou složku a max. 40% pohyblivou složku dle hodnocení. Příplatky a bonusy jsou nad rámec.\n\nZaokrouhlení: mezivýpočty na 2 desetinná místa, celková hrubá mzda na celé Kč. PN a neplacené jsou evidenční (náhradu mzdy při PN lze doplnit).")