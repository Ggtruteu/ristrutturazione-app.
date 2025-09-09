# ristrutturazione-app.
import streamlit as st
import pandas as pd
import io

# ==========================
# MATERIALI
# ==========================
materials = {
    "cemento": {"densita": 2400, "costo_per_m3": 100},   # kg/m3, €/m3
    "legno": {"densita": 700, "costo_per_m3": 150},
    "mattoni": {"peso_per_pezzo": 2.5, "costo_per_pezzo": 1.2},
    "vetro": {"peso_per_m2": 25, "costo_per_m2": 50}
}

# ==========================
# FUNZIONI UTILI
# ==========================
def calcola_volume(tipo, d):
    if tipo in ["muro", "tetto", "pavimento"]:
        return d.get("lunghezza", 0.0) * d.get("altezza", 0.0) * d.get("spessore", 1.0)
    elif tipo in ["porta", "finestra"]:
        return d.get("larghezza", 0.0) * d.get("altezza", 0.0)
    return 0.0

def calcola_peso_parte(parte):
    mat = materials[parte["materiale"]]
    vol = calcola_volume(parte["tipo"], parte["dimensioni"])
    q = parte["quantita"]
    if "densita" in mat:
        return vol * mat["densita"] * q
    if "peso_per_m2" in mat:
        return vol * mat["peso_per_m2"] * q
    if "peso_per_pezzo" in mat:
        return mat["peso_per_pezzo"] * q
    return 0.0

def calcola_costo_parte(parte):
    mat = materials[parte["materiale"]]
    vol = calcola_volume(parte["tipo"], parte["dimensioni"])
    q = parte["quantita"]
    if "costo_per_m3" in mat:
        return vol * mat["costo_per_m3"] * q
    if "costo_per_m2" in mat:
        return vol * mat["costo_per_m2"] * q
    if "costo_per_pezzo" in mat:
        return mat["costo_per_pezzo"] * q
    return 0.0

# ==========================
# STREAMLIT - STATE
# ==========================
if "parts" not in st.session_state:
    st.session_state.parts = []

st.title("🏗️ Software Ristrutturazione (Streamlit)")

with st.sidebar:
    st.header("Aggiungi parte")
    nome = st.text_input("Nome parte", key="nome")
    tipo = st.selectbox("Tipo", ["muro", "tetto", "pavimento", "porta", "finestra"], key="tipo")
    materiale = st.selectbox("Materiale", list(materials.keys()), key="materiale")
    quantita = st.number_input("Quantità", min_value=1, value=1, step=1, key="quantita")

    # dimensioni dinamiche
    if tipo in ["muro", "tetto", "pavimento"]:
        lunghezza = st.number_input("Lunghezza (m)", min_value=0.0, value=1.0, key="lunghezza")
        altezza = st.number_input("Altezza (m)", min_value=0.0, value=1.0, key="altezza")
        spessore = st.number_input("Spessore (m)", min_value=0.0, value=0.1, key="spessore")
        dimensioni = {"lunghezza": lunghezza, "altezza": altezza, "spessore": spessore}
    else:
        larghezza = st.number_input("Larghezza (m)", min_value=0.0, value=1.0, key="larghezza")
        altezza = st.number_input("Altezza (m)", min_value=0.0, value=2.0, key="altezza_pf")
        dimensioni = {"larghezza": larghezza, "altezza": altezza}

    if st.button("➕ Aggiungi parte"):
        new_id = len(st.session_state.parts) + 1
        parte = {
            "id": new_id,
            "nome": nome if nome else f"{tipo} {new_id}",
            "tipo": tipo,
            "materiale": materiale,
            "dimensioni": dimensioni,
            "quantita": int(quantita)
        }
        st.session_state.parts.append(parte)
        st.success(f"✅ {parte['nome']} aggiunta!")

st.header("Elenco parti")
if not st.session_state.parts:
    st.info("Nessuna parte aggiunta.")
else:
    # prepara tabella
    rows = []
    for p in st.session_state.parts:
        peso = calcola_peso_parte(p)
        costo = calcola_costo_parte(p)
        d = p["dimensioni"]
        row = {
            "ID": p["id"],
            "Nome": p["nome"],
            "Tipo": p["tipo"],
            "Materiale": p["materiale"],
            "Quantità": p["quantita"],
            "Lunghezza": d.get("lunghezza", ""),
            "Larghezza": d.get("larghezza", ""),
            "Altezza": d.get("altezza", ""),
            "Spessore": d.get("spessore", ""),
            "Peso (kg)": round(peso, 2),
            "Costo (€)": round(costo, 2)
        }
        rows.append(row)
    df = pd.DataFrame(rows).sort_values("ID")
    st.dataframe(df, use_container_width=True)

    totale_peso = df["Peso (kg)"].sum()
    totale_costo = df["Costo (€)"].sum()
    st.write("---")
    col1, col2 = st.columns(2)
    col1.metric("Peso totale (kg)", f"{totale_peso:.2f}")
    col2.metric("Costo totale (€)", f"{totale_costo:.2f}")

    # download CSV
    csv = df.to_csv(index=False)
    st.download_button("⬇️ Esporta CSV", csv, file_name="ristrutturazione_report.csv", mime="text/csv")

    # reset / rimuovi
    if st.button("🧹 Cancella tutte le parti"):
        st.session_state.parts = []
        st.experimental_rerun()