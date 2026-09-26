import json
from html import escape as esc

import pandas as pd
import streamlit as st
from snowflake.snowpark.context import get_active_session

try:
    st.set_page_config(
        page_title="Civic Grievance Triager",
        page_icon="🏛️",
        layout="wide",
        initial_sidebar_state="expanded",
    )
except Exception:
    pass

session = get_active_session()

# App session has no default database/schema -> set it, and fully qualify names anyway
for fn, arg in (
    (session.use_warehouse, "CIVIC_WH"),
    (session.use_database, "CIVIC_TRIAGER"),
    (session.use_schema, "CORE"),
):
    try:
        fn(arg)
    except Exception:
        pass

TBL = "CIVIC_TRIAGER.CORE.COMPLAINTS"
PROC = "CIVIC_TRIAGER.CORE.PROCESS_COMPLAINT"
SVC = "CIVIC_TRIAGER.CORE.COMPLAINT_SEARCH"

LANGS = {"English": "en", "हिन्दी (Hindi)": "hi", "ગુજરાતી (Gujarati)": "gu"}
LANG_NAMES = {"en": "English", "hi": "Hindi", "gu": "Gujarati"}
STATUSES = ["New", "In Progress", "Resolved"]

SAMPLES = {
    "ગુજરાતી sample": (
        "ગુજરાતી (Gujarati)",
        "અમારી સોસાયટી પાસે ગટર ઊભરાઈ રહી છે અને ગંદું પાણી રસ્તા પર ફેલાઈ ગયું છે. "
        "બાળકો બીમાર પડી રહ્યા છે, કૃપા કરીને તાત્કાલિક મદદ કરો.",
    ),
    "हिन्दी sample": (
        "हिन्दी (Hindi)",
        "हमारे इलाके में पिछले तीन दिनों से पानी नहीं आ रहा है। "
        "बुजुर्गों और बच्चों को बहुत परेशानी हो रही है, कृपया जल्दी ठीक करवाइए।",
    ),
    "English sample": (
        "English",
        "The streetlight near the bus stop has been off for a week. The road is completely "
        "dark at night and women feel unsafe walking home.",
    ),
}

CAT_ICON = {
    "roads": "🛣️",
    "water": "💧",
    "electricity": "⚡",
    "garbage": "🗑️",
    "drainage": "🚰",
    "other": "📌",
}

# score -> (name, badge background, badge text, accent)
URGENCY_LEVELS = {
    5: ("Critical", "#fee2e2", "#991b1b", "#dc2626"),
    4: ("High", "#ffedd5", "#9a3412", "#ea580c"),
    3: ("Medium", "#fef9c3", "#854d0e", "#ca8a04"),
    2: ("Low", "#dcfce7", "#166534", "#16a34a"),
    1: ("Low", "#dcfce7", "#166534", "#16a34a"),
}

STATUS_COLORS = {
    "New": ("#e0e7ff", "#3730a3"),
    "In Progress": ("#fef3c7", "#92400e"),
    "Resolved": ("#d1fae5", "#065f46"),
}

CSS = """
<style>
.block-container, [data-testid="stMainBlockContainer"] {padding-top: 1.2rem; max-width: 1350px;}
.hero {background: linear-gradient(120deg, #0f766e 0%, #1e40af 100%); color: #ffffff; padding: 22px 28px; border-radius: 18px; margin-bottom: 18px;}
.hero-title {font-size: 1.9rem; font-weight: 800; color: #ffffff; line-height: 1.2;}
.hero-sub {color: rgba(255,255,255,0.88); margin-top: 6px; font-size: 1rem;}
.chip {display: inline-block; background: rgba(255,255,255,0.16); color: #ffffff; border: 1px solid rgba(255,255,255,0.35); padding: 3px 12px; border-radius: 999px; font-size: 0.8rem; margin: 12px 6px 0 0;}
.card {border: 1px solid rgba(127,127,127,0.25); border-radius: 14px; padding: 16px 18px; background: rgba(127,127,127,0.05); margin-bottom: 12px;}
.card-title {font-weight: 700; font-size: 1.05rem; margin-bottom: 8px;}
.section-title {font-weight: 700; font-size: 1.15rem; margin: 16px 0 8px 0;}
.muted {opacity: 0.7; font-size: 0.85rem;}
.quote {font-size: 0.95rem; line-height: 1.55;}
.kpi-row {display: grid; grid-template-columns: repeat(4, 1fr); gap: 12px; margin: 4px 0 14px 0;}
.kpi {border: 1px solid rgba(127,127,127,0.25); border-radius: 14px; padding: 14px 18px; background: rgba(127,127,127,0.05);}
.kpi-label {font-size: 0.75rem; opacity: 0.7; text-transform: uppercase; letter-spacing: 0.05em;}
.kpi-value {font-size: 2rem; font-weight: 800; line-height: 1.15;}
.kpi-sub {font-size: 0.8rem; opacity: 0.65;}
.badge {display: inline-block; padding: 3px 11px; border-radius: 999px; font-size: 0.78rem; font-weight: 600; margin: 0 6px 6px 0;}
.mini {border-left: 4px solid #1e40af; border-radius: 8px; padding: 8px 12px; margin: 8px 0; background: rgba(127,127,127,0.06);}
.mini-meta {font-size: 0.78rem; opacity: 0.7; margin-bottom: 4px;}
.step {display: flex; gap: 12px; align-items: flex-start; margin: 12px 0;}
.step-n {min-width: 28px; height: 28px; border-radius: 50%; background: #1e40af; color: #ffffff; display: flex; align-items: center; justify-content: center; font-weight: 700; font-size: 0.85rem;}
.step-t {font-weight: 600;}
.step-d {font-size: 0.82rem; opacity: 0.7;}
@media (max-width: 900px) {.kpi-row {grid-template-columns: repeat(2, 1fr);}}
</style>
"""


# =====================================================================
# UI HELPERS
# =====================================================================
def show_html(s):
    st.markdown(s, unsafe_allow_html=True)


def badge(text, bg, fg):
    return f'<span class="badge" style="background:{bg};color:{fg};">{esc(str(text))}</span>'


def urgency_level(u):
    """Return (score, name, bg, fg, accent) or None if not scored yet."""
    try:
        n = int(float(u))
    except (TypeError, ValueError):
        return None
    if n not in URGENCY_LEVELS:
        return None
    return (n,) + URGENCY_LEVELS[n]


def urgency_badge(u):
    lvl = urgency_level(u)
    if not lvl:
        return badge("Urgency pending", "#e5e7eb", "#374151")
    n, name, bg, fg, _ = lvl
    return badge(f"{name} · {n}/5", bg, fg)


def category_badge(cat):
    if not isinstance(cat, str) or not cat:
        cat = "other"
    return badge(f"{CAT_ICON.get(cat, '📌')} {cat.title()}", "#dbeafe", "#1e3a8a")


def status_badge(status):
    if not isinstance(status, str) or not status:
        status = "New"
    bg, fg = STATUS_COLORS.get(status, ("#e5e7eb", "#374151"))
    return badge(status, bg, fg)


def lang_badge(name):
    return badge(f"🌐 {name}", "#f3e8ff", "#6b21a8")


def priority_label(u):
    lvl = urgency_level(u)
    if not lvl:
        return "⚪ Pending"
    icon = {5: "🔴", 4: "🟠", 3: "🟡"}.get(lvl[0], "🟢")
    return f"{icon} {lvl[1]}"


def sentiment_label(s):
    try:
        s = float(s)
    except (TypeError, ValueError):
        return "Sentiment -"
    if pd.isna(s):
        return "Sentiment -"
    if s <= -0.3:
        mood = "😟 Distressed"
    elif s >= 0.3:
        mood = "🙂 Positive"
    else:
        mood = "😐 Neutral"
    return f"{mood} ({s:.2f})"


def kpi(label, value, sub, color=None):
    style = f' style="color:{color};"' if color else ""
    return (
        f'<div class="kpi"><div class="kpi-label">{esc(label)}</div>'
        f'<div class="kpi-value"{style}>{esc(str(value))}</div>'
        f'<div class="kpi-sub">{esc(sub)}</div></div>'
    )


def step(n, title, desc):
    return (
        f'<div class="step"><div class="step-n">{n}</div><div>'
        f'<div class="step-t">{esc(title)}</div><div class="step-d">{esc(desc)}</div></div></div>'
    )


def hero():
    show_html(
        '<div class="hero">'
        '<div class="hero-title">🏛️ Ahmedabad Civic Grievance Triager</div>'
        '<div class="hero-sub">Complaints in Gujarati, Hindi or English, translated, sorted and prioritised automatically with Snowflake Cortex.</div>'
        '<span class="chip">ગુજરાતી</span><span class="chip">हिन्दी</span><span class="chip">English</span>'
        '<span class="chip">❄️ Runs entirely in Snowflake</span>'
        "</div>"
    )


# =====================================================================
# CORTEX SEARCH HELPERS
# =====================================================================
def search_similar(query_text, ward=None, category=None, exclude_id=None, limit=5):
    """Semantic search over past complaints via Cortex Search."""
    conds = []
    if ward:
        conds.append({"@eq": {"ward": ward}})
    if category:
        conds.append({"@eq": {"category": category}})

    payload = {
        "query": (query_text or "").replace("$", "")[:1000],
        "columns": [
            "complaint_id", "ward", "category", "urgency_score",
            "status", "clerk_summary", "submitted_at",
        ],
        "limit": limit + 1,
    }
    if len(conds) == 1:
        payload["filter"] = conds[0]
    elif len(conds) > 1:
        payload["filter"] = {"@and": conds}

    body = json.dumps(payload, ensure_ascii=False)
    raw = session.sql(
        f"SELECT SNOWFLAKE.CORTEX.SEARCH_PREVIEW('{SVC}', $${body}$$) AS R"
    ).collect()[0]["R"]
    data = json.loads(raw) if isinstance(raw, str) else raw

    results = []
    for r in data.get("results", []):
        r = {k.lower(): v for k, v in r.items()}
        if exclude_id is not None and str(r.get("complaint_id")) == str(exclude_id):
            continue
        results.append(r)
    return results[:limit]


def show_similar(results, empty_msg):
    if not results:
        st.caption(empty_msg)
        return
    parts = []
    for r in results:
        lvl = urgency_level(r.get("urgency_score"))
        accent = lvl[4] if lvl else "#6b7280"
        meta = f"#{r.get('complaint_id')} · {r.get('ward')} · {r.get('submitted_at') or ''}"
        parts.append(
            f'<div class="mini" style="border-left-color:{accent};">'
            f'<div class="mini-meta">{esc(meta)}</div>'
            f"{category_badge(r.get('category'))}{urgency_badge(r.get('urgency_score'))}{status_badge(r.get('status'))}"
            f"<div>{esc(str(r.get('clerk_summary') or ''))}</div>"
            "</div>"
        )
    show_html("".join(parts))


# =====================================================================
# CITIZEN PORTAL
# =====================================================================
def citizen_portal():
    left, right = st.columns([3, 2], gap="large")

    with right:
        show_html(
            '<div class="card"><div class="card-title">How your complaint is handled</div>'
            + step(1, "Understood in your language", "Gujarati and Hindi are translated to English.")
            + step(2, "Sent to the right department", "Roads, water, electricity, garbage, drainage or other.")
            + step(3, "Scored for urgency", "1 = minor issue, 5 = immediate safety or health risk.")
            + step(4, "Summarised for the clerk", "One clear line so staff can act faster.")
            + step(5, "Checked against past reports", "Spots repeat problems in the same ward.")
            + "</div>"
        )

    with left:
        show_html('<div class="section-title" style="margin-top:0;">📝 File a complaint</div>')
        st.caption("Write in Gujarati, Hindi or English, or try a sample:")

        cols = st.columns(3)
        for col, (btn_label, (lang_label, sample_text)) in zip(cols, SAMPLES.items()):
            if col.button(btn_label, use_container_width=True):
                st.session_state["lang"] = lang_label
                st.session_state["text"] = sample_text

        wards = [
            r["WARD"]
            for r in session.sql(
                f"SELECT DISTINCT ward AS WARD FROM {TBL} WHERE ward IS NOT NULL ORDER BY ward"
            ).collect()
        ]

        with st.form("complaint_form"):
            c1, c2 = st.columns(2)
            with c1:
                name = st.text_input("Your name", key="name")
            with c2:
                if wards:
                    ward = st.selectbox("Ward", wards, key="ward")
                else:
                    ward = st.text_input("Ward", key="ward")
            lang_label = st.selectbox("Language", list(LANGS.keys()), key="lang")
            text = st.text_area(
                "Describe the problem",
                height=140,
                key="text",
                placeholder="Example: The drain near our society is overflowing onto the road...",
            )
            submitted = st.form_submit_button("Submit complaint", type="primary", use_container_width=True)

        if not submitted:
            return
        if not name.strip() or not text.strip():
            st.error("Please fill in your name and the complaint.")
            return

        lang = LANGS[lang_label]
        err = ""
        with st.spinner("Submitting and analysing your complaint..."):
            session.sql(
                f"INSERT INTO {TBL} (submitted_at, citizen_name, ward, original_language, original_text) "
                "VALUES (CURRENT_TIMESTAMP(), ?, ?, ?, ?)",
                params=[name.strip(), ward, lang, text.strip()],
            ).collect()

            new_id = int(
                session.sql(
                    f"SELECT MAX(complaint_id) AS ID FROM {TBL} "
                    "WHERE citizen_name = ? AND original_text = ?",
                    params=[name.strip(), text.strip()],
                ).collect()[0]["ID"]
            )

            try:
                session.sql(f"CALL {PROC}({new_id})").collect()
                analysed = True
            except Exception as e:
                analysed = False
                err = str(e)

        if not analysed:
            st.warning(f"Complaint #{new_id} saved. Automatic analysis failed; it will be retried.")
            with st.expander("Error details"):
                st.code(err)
            return

        row = session.sql(
            "SELECT category, urgency_score, clerk_summary, translated_text "
            f"FROM {TBL} WHERE complaint_id = {new_id}"
        ).collect()[0]

        show_html(
            '<div class="card" style="border-left:6px solid #16a34a;">'
            f'<div class="card-title">✅ Complaint #{new_id} registered</div>'
            f"{category_badge(row['CATEGORY'])}{urgency_badge(row['URGENCY_SCORE'])}"
            f"{status_badge('New')}{lang_badge(LANG_NAMES[lang])}"
            '<div class="muted" style="margin-top:8px;">Summary for the clerk</div>'
            f'<div class="quote">{esc(row["CLERK_SUMMARY"] or "")}</div>'
            "</div>"
        )
        if lang != "en":
            with st.expander("English translation"):
                st.write(row["TRANSLATED_TEXT"])

        show_html('<div class="section-title">🔁 Similar reports already filed in your ward</div>')
        try:
            res = search_similar(
                row["TRANSLATED_TEXT"], ward=ward, category=row["CATEGORY"], exclude_id=new_id, limit=3
            )
            show_similar(res, "None found. Yours is the first report of this kind in this ward.")
        except Exception as e:
            st.caption(f"Search unavailable: {e}")


# =====================================================================
# ADMIN DASHBOARD
# =====================================================================
def highlight(row):
    p = str(row["Priority"])
    if p.startswith("🔴"):
        color = "background-color: rgba(220, 38, 38, 0.18)"
    elif p.startswith("🟠"):
        color = "background-color: rgba(234, 88, 12, 0.14)"
    else:
        color = ""
    return [color] * len(row)


def admin_dashboard():
    h1, h2 = st.columns([5, 1])
    with h1:
        show_html('<div class="section-title" style="font-size:1.4rem;margin-top:0;">📊 Triage dashboard</div>')
    with h2:
        st.button("🔄 Refresh", use_container_width=True)  # any click reruns the app and reloads data

    df = session.sql(f"""
        SELECT complaint_id, submitted_at, citizen_name, ward, original_language,
               category, urgency_score, sentiment_score, clerk_summary, status,
               original_text, translated_text
          FROM {TBL}
         ORDER BY urgency_score DESC NULLS LAST, submitted_at DESC
    """).to_pandas()

    df["URGENCY_SCORE"] = pd.to_numeric(df["URGENCY_SCORE"], errors="coerce")
    df["SENTIMENT_SCORE"] = pd.to_numeric(df["SENTIMENT_SCORE"], errors="coerce")
    df["SUBMITTED_AT"] = pd.to_datetime(df["SUBMITTED_AT"], errors="coerce")
    df["LANGUAGE"] = (
        df["ORIGINAL_LANGUAGE"].astype(str).str.lower().map(LANG_NAMES).fillna(df["ORIGINAL_LANGUAGE"])
    )
    df["STATUS"] = df["STATUS"].fillna("New")

    # ---- KPI cards ----
    total = len(df)
    urgent = int((df["URGENCY_SCORE"] >= 4).sum())
    critical = int((df["URGENCY_SCORE"] >= 5).sum())
    open_n = int((df["STATUS"] != "Resolved").sum())
    show_html(
        '<div class="kpi-row">'
        + kpi("Total complaints", total, f"across {df['WARD'].nunique()} wards")
        + kpi("Urgent (4-5)", urgent, f"{critical} critical", color="#dc2626")
        + kpi("Open", open_n, f"{total - open_n} resolved", color="#1e40af")
        + kpi("Languages", int(df["LANGUAGE"].nunique()), "Gujarati · Hindi · English")
        + "</div>"
    )

    # ---- Filters ----
    with st.container(border=True):
        f1, f2, f3 = st.columns([2, 1, 1])
        cats = sorted(df["CATEGORY"].dropna().unique().tolist())
        sel_cats = f1.multiselect("Category", cats, default=cats)
        ward_opts = ["All wards"] + sorted(df["WARD"].dropna().unique().tolist())
        sel_ward = f2.selectbox("Ward", ward_opts)
        min_urg = f3.slider("Minimum urgency", 1, 5, 1)

    queue = df[df["CATEGORY"].isin(sel_cats or cats) & (df["URGENCY_SCORE"].fillna(0) >= min_urg)]
    if sel_ward != "All wards":
        queue = queue[queue["WARD"] == sel_ward]

    # ---- Charts ----
    ch1, ch2 = st.columns(2)
    with ch1:
        with st.container(border=True):
            st.markdown("**Complaints by category**")
            if len(queue):
                st.bar_chart(queue.groupby("CATEGORY").size().rename("Complaints"), height=260)
            else:
                st.caption("No complaints match these filters.")
    with ch2:
        with st.container(border=True):
            st.markdown("**Urgent complaints (4-5) by ward**")
            urgent_by_ward = queue[queue["URGENCY_SCORE"] >= 4].groupby("WARD").size().rename("Urgent")
            if len(urgent_by_ward):
                st.bar_chart(urgent_by_ward, height=260)
            else:
                st.caption("No urgent complaints in this view.")

    # ---- Queue table ----
    show_html('<div class="section-title">📋 Queue, most urgent first</div>')
    table = pd.DataFrame({
        "ID": queue["COMPLAINT_ID"],
        "Priority": queue["URGENCY_SCORE"].apply(priority_label),
        "Category": queue["CATEGORY"].apply(
            lambda c: f"{CAT_ICON.get(c, '📌')} {c}" if isinstance(c, str) else "-"
        ),
        "Ward": queue["WARD"],
        "Language": queue["LANGUAGE"],
        "Summary": queue["CLERK_SUMMARY"],
        "Status": queue["STATUS"],
        "Submitted": queue["SUBMITTED_AT"].dt.strftime("%d %b, %H:%M"),
    })
    st.dataframe(
        table.style.apply(highlight, axis=1),
        hide_index=True,
        use_container_width=True,
        height=min(460, 40 + 35 * max(len(table), 1)),
    )

    # ---- Detail ----
    show_html('<div class="section-title">🔎 Open a complaint</div>')
    if queue.empty:
        st.caption("No complaints match these filters.")
        return

    labels = {}
    for _, r in queue.iterrows():
        s = r["CLERK_SUMMARY"] if isinstance(r["CLERK_SUMMARY"], str) else "Not analysed yet"
        labels[int(r["COMPLAINT_ID"])] = f"#{int(r['COMPLAINT_ID'])} · {priority_label(r['URGENCY_SCORE'])} · {s[:70]}"
    cid = st.selectbox("Pick a complaint", list(labels.keys()), format_func=lambda i: labels.get(i, str(i)))
    rec = df[df["COMPLAINT_ID"] == cid].iloc[0]

    summary = rec["CLERK_SUMMARY"] if isinstance(rec["CLERK_SUMMARY"], str) else "Not analysed yet"
    orig = rec["ORIGINAL_TEXT"] if isinstance(rec["ORIGINAL_TEXT"], str) else ""
    trans = rec["TRANSLATED_TEXT"] if isinstance(rec["TRANSLATED_TEXT"], str) else ""
    when = rec["SUBMITTED_AT"].strftime("%d %b %Y, %H:%M") if pd.notna(rec["SUBMITTED_AT"]) else "-"
    lvl = urgency_level(rec["URGENCY_SCORE"])
    accent = lvl[4] if lvl else "#6b7280"

    d1, d2 = st.columns([3, 2], gap="large")
    with d1:
        detail = (
            f'<div class="card" style="border-left:6px solid {accent};">'
            f'<div class="card-title">#{int(cid)} · {esc(summary)}</div>'
            f"{category_badge(rec['CATEGORY'])}{urgency_badge(rec['URGENCY_SCORE'])}"
            f"{status_badge(rec['STATUS'])}{lang_badge(rec['LANGUAGE'])}"
            f'<div class="muted" style="margin-top:4px;">👤 {esc(str(rec["CITIZEN_NAME"]))} · 📍 {esc(str(rec["WARD"]))} · 🕒 {esc(when)} · {esc(sentiment_label(rec["SENTIMENT_SCORE"]))}</div>'
            f'<div class="muted" style="margin-top:14px;">Original ({esc(str(rec["LANGUAGE"]))})</div>'
            f'<div class="quote">{esc(orig)}</div>'
        )
        if trans and trans != orig:
            detail += (
                '<div class="muted" style="margin-top:12px;">English translation</div>'
                f'<div class="quote">{esc(trans)}</div>'
            )
        detail += "</div>"
        show_html(detail)

    with d2:
        with st.container(border=True):
            st.markdown("**🔁 Has this happened before?**")
            st.caption("Semantic search over past complaints with Cortex Search")
            scope = st.radio(
                "Search scope", ["Same ward + category", "Whole city"], horizontal=True, key=f"scope_{cid}"
            )
            q = trans or orig
            try:
                if scope == "Same ward + category":
                    res = search_similar(q, ward=rec["WARD"], category=rec["CATEGORY"], exclude_id=cid)
                    show_similar(res, "No earlier complaints of this kind in this ward.")
                else:
                    res = search_similar(q, exclude_id=cid)
                    show_similar(res, "No similar complaints found.")
            except Exception as e:
                st.caption(f"Search unavailable: {e}")

        with st.container(border=True):
            st.markdown("**✏️ Update status**")
            cur = rec["STATUS"] if rec["STATUS"] in STATUSES else "New"
            new_status = st.selectbox(
                "Status", STATUSES, index=STATUSES.index(cur), key=f"status_{cid}", label_visibility="collapsed"
            )
            if st.button("Save status", type="primary", use_container_width=True, key=f"save_{cid}"):
                session.sql(
                    f"UPDATE {TBL} SET status = ? WHERE complaint_id = ?",
                    params=[new_status, int(cid)],
                ).collect()
                st.rerun()


# =====================================================================
# PAGE
# =====================================================================
st.markdown(CSS, unsafe_allow_html=True)

with st.sidebar:
    show_html(
        '<div class="card-title" style="font-size:1.25rem;">🏛️ Grievance Triager</div>'
        '<div class="muted">Hack Days Ahmedabad · built on Snowflake</div>'
    )
    st.write("")
    view = st.radio("Go to", ["📝 Citizen portal", "📊 Admin dashboard"], key="view")
    st.divider()
    st.markdown("**What runs under the hood**")
    st.markdown(
        "- **Translate:** Cortex `TRANSLATE` (Hindi), `COMPLETE` (Gujarati)\n"
        "- **Category:** `CLASSIFY_TEXT`\n"
        "- **Sentiment:** `SENTIMENT`\n"
        "- **Urgency + summary:** `COMPLETE` (claude-sonnet-4-5)\n"
        "- **Repeat issues:** Cortex Search"
    )

hero()

if view == "📝 Citizen portal":
    citizen_portal()
else:
    admin_dashboard()
