# OrtoManager
import sqlite3
from datetime import date, datetime
from pathlib import Path

import pandas as pd
import streamlit as st


# =========================================================
# CONFIGURAÇÃO
# =========================================================

st.set_page_config(
    page_title="OrtoManager",
    page_icon="👁️",
    layout="wide",
    initial_sidebar_state="expanded",
)

BASE_DIR = Path(__file__).resolve().parent
DB_PATH = BASE_DIR / "ortomanager.db"

STATUS_CIRCUITO = [
    "Chegou",
    "Aguarda Ortóptica",
    "Em exames",
    "Aguarda consulta médica",
    "Concluído",
]

TIPOS_EXAME = [
    "Avaliação ortóptica",
    "OCT",
    "Campimetria",
    "Biometria",
    "Retinografia",
    "Topografia",
    "Paquimetria",
    "Outro",
]

STATUS_EXAME = [
    "Por realizar",
    "Em curso",
    "Realizado",
    "Repetido",
    "Não realizado",
]

ESTADO_EQUIPAMENTO = [
    "Disponível",
    "Em manutenção",
    "Avariado",
]

ESTADO_MELHORIA = [
    "Por iniciar",
    "Em curso",
    "Concluída",
]


# =========================================================
# ESTILO
# =========================================================

st.markdown(
    """
    <style>

    .block-container {
        padding-top: 1.3rem;
        padding-bottom: 2rem;
    }

    [data-testid="stMetric"] {
        border: 1px solid rgba(128,128,128,0.20);
        padding: 14px;
        border-radius: 12px;
    }

    </style>
    """,
    unsafe_allow_html=True,
)


# =========================================================
# BASE DE DADOS
# =========================================================

def get_conn():
    return sqlite3.connect(DB_PATH, check_same_thread=False)


def execute(sql, params=()):
    conn = get_conn()
    cursor = conn.cursor()

    cursor.execute(sql, params)

    conn.commit()

    last_id = cursor.lastrowid

    conn.close()

    return last_id


def query(sql, params=()):

    conn = get_conn()

    df = pd.read_sql_query(
        sql,
        conn,
        params=params
    )

    conn.close()

    return df


def init_db():

    conn = get_conn()

    cursor = conn.cursor()

    # UTENTES

    cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS utentes (

            id INTEGER PRIMARY KEY AUTOINCREMENT,

            codigo TEXT NOT NULL UNIQUE,

            hora_chegada TEXT NOT NULL,

            estado TEXT NOT NULL,

            observacao TEXT DEFAULT ''

        )
        """
    )

    # EXAMES

    cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS exames (

            id INTEGER PRIMARY KEY AUTOINCREMENT,

            utente_codigo TEXT NOT NULL,

            tipo TEXT NOT NULL,

            lateralidade TEXT NOT NULL,

            estado TEXT NOT NULL,

            qualidade TEXT DEFAULT '',

            motivo_repeticao TEXT DEFAULT '',

            criado_em TEXT NOT NULL

        )
        """
    )

    # QUALIDADE

    cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS qualidade (

            id INTEGER PRIMARY KEY AUTOINCREMENT,

            data TEXT NOT NULL,

            utente_codigo TEXT NOT NULL,

            identificacao_ok INTEGER NOT NULL,

            lateralidade_ok INTEGER NOT NULL,

            protocolo_ok INTEGER NOT NULL,

            qualidade_ok INTEGER NOT NULL,

            observacao TEXT DEFAULT ''

        )
        """
    )

    # EQUIPAMENTOS

    cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS equipamentos (

            id INTEGER PRIMARY KEY AUTOINCREMENT,

            nome TEXT NOT NULL,

            localizacao TEXT DEFAULT '',

            estado TEXT NOT NULL,

            ultima_manutencao TEXT DEFAULT '',

            proxima_manutencao TEXT DEFAULT '',

            observacao TEXT DEFAULT ''

        )
        """
    )

    # MELHORIAS

    cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS melhorias (

            id INTEGER PRIMARY KEY AUTOINCREMENT,

            problema TEXT NOT NULL,

            acao TEXT NOT NULL,

            responsavel TEXT DEFAULT '',

            prazo TEXT DEFAULT '',

            estado TEXT NOT NULL,

            resultado TEXT DEFAULT ''

        )
        """
    )

    conn.commit()

    conn.close()


init_db()


# =========================================================
# FUNÇÕES AUXILIARES
# =========================================================

def codigos_utentes():

    df = query(
        """
        SELECT codigo
        FROM utentes
        ORDER BY id DESC
        """
    )

    if df.empty:
        return []

    return df["codigo"].tolist()


def tempo_espera_minutos(hora_chegada):

    try:

        inicio = datetime.strptime(
            hora_chegada,
            "%Y-%m-%d %H:%M"
        )

        minutos = (
            datetime.now() - inicio
        ).total_seconds() / 60

        return max(
            0,
            int(minutos)
        )

    except Exception:

        return 0


# =========================================================
# SIDEBAR
# =========================================================

with st.sidebar:

    st.markdown(
        "# 👁️ OrtoManager"
    )

    st.caption(
        "Gestão Operacional • Qualidade • Melhoria Contínua"
    )

    st.divider()

    pagina = st.radio(
        "Menu",
        [
            "Dashboard",
            "Circuito do Utente",
            "Exames",
            "Qualidade",
            "Equipamentos",
            "Melhoria Contínua",
        ],
    )

    st.divider()

    st.caption(
        "Protótipo demonstrativo"
    )


# =========================================================
# DASHBOARD
# =========================================================

if pagina == "Dashboard":

    st.title(
        "👁️ OrtoManager"
    )

    st.subheader(
        "Dashboard do Serviço de Ortóptica"
    )

    utentes = query(
        """
        SELECT *
        FROM utentes
        ORDER BY id DESC
        """
    )

    exames = query(
        """
        SELECT *
        FROM exames
        ORDER BY id DESC
        """
    )

    equipamentos = query(
        """
        SELECT *
        FROM equipamentos
        ORDER BY id DESC
        """
    )

    hoje = date.today().isoformat()

    if not utentes.empty:

        utentes_hoje = utentes[
            utentes["hora_chegada"].str.startswith(
                hoje
            )
        ]

    else:

        utentes_hoje = utentes

    if not exames.empty:

        exames_hoje = exames[
            exames["criado_em"].str.startswith(
                hoje
            )
        ]

    else:

        exames_hoje = exames

    if not utentes_hoje.empty:

        ativos = utentes_hoje[
            utentes_hoje["estado"] != "Concluído"
        ]

    else:

        ativos = utentes_hoje

    if not exames_hoje.empty:

        repetidos = exames_hoje[
            exames_hoje["estado"] == "Repetido"
        ]

    else:

        repetidos = exames_hoje

    if not equipamentos.empty:

        indisponiveis = equipamentos[
            equipamentos["estado"] != "Disponível"
        ]

    else:

        indisponiveis = equipamentos

    if not ativos.empty:

        esperas = [
            tempo_espera_minutos(x)
            for x in ativos["hora_chegada"]
        ]

    else:

        esperas = []

    if esperas:

        espera_media = round(
            sum(esperas) / len(esperas)
        )

    else:

        espera_media = 0

    c1, c2, c3, c4, c5 = st.columns(5)

    c1.metric(
        "Utentes hoje",
        len(utentes_hoje)
    )

    c2.metric(
        "Exames hoje",
        len(exames_hoje)
    )

    c3.metric(
        "Espera média",
        f"{espera_media} min"
    )

    c4.metric(
        "Exames repetidos",
        len(repetidos)
    )

    c5.metric(
        "Equipamentos indisponíveis",
        len(indisponiveis)
    )

    st.divider()

    st.subheader(
        "Circuito atual"
    )

    if ativos.empty:

        st.success(
            "Não existem utentes em circuito."
        )

    else:

        tabela = ativos[
            [
                "codigo",
                "hora_chegada",
                "estado",
                "observacao"
            ]
        ].copy()

        tabela["espera"] = tabela[
            "hora_chegada"
        ].apply(
            tempo_espera_minutos
        )

        tabela.columns = [
            "Utente",
            "Hora chegada",
            "Estado",
            "Observação",
            "Espera (min)"
        ]

        st.dataframe(
            tabela,
            use_container_width=True,
            hide_index=True
        )


# =========================================================
# CIRCUITO DO UTENTE
# =========================================================

elif pagina == "Circuito do Utente":

    st.title(
        "👥 Circuito do Utente"
    )

    tab1, tab2, tab3 = st.tabs(
        [
            "Circuito atual",
            "Adicionar utente",
            "Atualizar estado"
        ]
    )

    # ----------------------------

    with tab1:

        utentes = query(
            """
            SELECT *
            FROM utentes
            ORDER BY id DESC
            """
        )

        if utentes.empty:

            st.info(
                "Ainda não existem utentes."
            )

        else:

            tabela = utentes.copy()

            tabela["espera"] = tabela[
                "hora_chegada"
            ].apply(
                tempo_espera_minutos
            )

            tabela = tabela[
                [
                    "codigo",
                    "hora_chegada",
                    "estado",
                    "espera",
                    "observacao"
                ]
            ]

            tabela.columns = [
                "Código",
                "Hora chegada",
                "Estado",
                "Espera (min)",
                "Observação"
            ]

            st.dataframe(
                tabela,
                use_container_width=True,
                hide_index=True
            )

    # ----------------------------

    with tab2:

        with st.form(
            "novo_utente",
            clear_on_submit=True
        ):

            codigo = st.text_input(
                "Código interno",
                placeholder="Ex.: UT-001"
            )

            estado = st.selectbox(
                "Estado inicial",
                STATUS_CIRCUITO
            )

            observacao = st.text_input(
                "Observação operacional"
            )

            guardar = st.form_submit_button(
                "Adicionar utente"
            )

            if guardar:

                codigo = codigo.strip().upper()

                if not codigo:

                    st.error(
                        "Introduz um código."
                    )

                else:

                    try:

                        execute(
                            """
                            INSERT INTO utentes
                            (
                                codigo,
                                hora_chegada,
                                estado,
                                observacao
                            )
                            VALUES (?, ?, ?, ?)
                            """,
                            (
                                codigo,
                                datetime.now().strftime(
                                    "%Y-%m-%d %H:%M"
                                ),
                                estado,
                                observacao
                            )
                        )

                        st.success(
                            "Utente adicionado."
                        )

                        st.rerun()

                    except sqlite3.IntegrityError:

                        st.error(
                            "Esse código já existe."
                        )

    # ----------------------------

    with tab3:

        codigos = codigos_utentes()

        if not codigos:

            st.info(
                "Ainda não existem utentes."
            )

        else:

            with st.form(
                "atualizar_utente"
            ):

                codigo = st.selectbox(
                    "Utente",
                    codigos
                )

                novo_estado = st.selectbox(
                    "Novo estado",
                    STATUS_CIRCUITO
                )

                observacao = st.text_input(
                    "Observação"
                )

                atualizar = st.form_submit_button(
                    "Atualizar"
                )

                if atualizar:

                    execute(
                        """
                        UPDATE utentes

                        SET
                            estado = ?,
                            observacao = ?

                        WHERE codigo = ?
                        """,
                        (
                            novo_estado,
                            observacao,
                            codigo
                        )
                    )

                    st.success(
                        "Estado atualizado."
                    )

                    st.rerun()


# =========================================================
# EXAMES
# =========================================================

elif pagina == "Exames":

    st.title(
        "🔬 Gestão de Exames"
    )

    tab1, tab2, tab3 = st.tabs(
        [
            "Exames",
            "Registar exame",
            "Atualizar exame"
        ]
    )

    with tab1:

        exames = query(
            """
            SELECT *
            FROM exames
            ORDER BY id DESC
            """
        )

        if exames.empty:

            st.info(
                "Ainda não existem exames."
            )

        else:

            tabela = exames[
                [
                    "id",
                    "utente_codigo",
                    "tipo",
                    "lateralidade",
                    "estado",
                    "qualidade",
                    "motivo_repeticao",
                    "criado_em",
                ]
            ].copy()

            tabela.columns = [
                "ID",
                "Utente",
                "Exame",
                "Olho",
                "Estado",
                "Qualidade",
                "Motivo repetição",
                "Data"
            ]

            st.dataframe(
                tabela,
                use_container_width=True,
                hide_index=True
            )

    with tab2:

        codigos = codigos_utentes()

        if not codigos:

            st.warning(
                "Primeiro adiciona um utente."
            )

        else:

            with st.form(
                "registar_exame"
            ):

                utente = st.selectbox(
                    "Utente",
                    codigos
                )

                tipo = st.selectbox(
                    "Exame",
                    TIPOS_EXAME
                )

                lateralidade = st.selectbox(
                    "Lateralidade",
                    [
                        "AO",
                        "OD",
                        "OE",
                        "N/A"
                    ]
                )

                estado = st.selectbox(
                    "Estado",
                    STATUS_EXAME
                )

                qualidade = st.selectbox(
                    "Qualidade",
                    [
                        "",
                        "Adequada",
                        "Aceitável",
                        "Inadequada"
                    ]
                )

                motivo = st.text_input(
                    "Motivo de repetição"
                )

                guardar = st.form_submit_button(
                    "Guardar exame"
                )

                if guardar:

                    execute(
                        """
                        INSERT INTO exames
                        (
                            utente_codigo,
                            tipo,
                            lateralidade,
                            estado,
                            qualidade,
                            motivo_repeticao,
                            criado_em
                        )

                        VALUES (?, ?, ?, ?, ?, ?, ?)
                        """,
                        (
                            utente,
                            tipo,
                            lateralidade,
                            estado,
                            qualidade,
                            motivo,
                            datetime.now().strftime(
                                "%Y-%m-%d %H:%M"
                            ),
                        )
                    )

                    st.success(
                        "Exame registado."
                    )

                    st.rerun()

    with tab3:

        exames = query(
            """
            SELECT
                id,
                utente_codigo,
                tipo,
                estado

            FROM exames

            ORDER BY id DESC
            """
        )

        if exames.empty:

            st.info(
                "Não existem exames."
            )

        else:

            opcoes = {}

            for row in exames.itertuples():

                nome = (
                    f"#{row.id} - "
                    f"{row.utente_codigo} - "
                    f"{row.tipo} - "
                    f"{row.estado}"
                )

                opcoes[nome] = row.id

            with st.form(
                "atualizar_exame"
            ):

                exame = st.selectbox(
                    "Exame",
                    list(opcoes.keys())
                )

                estado = st.selectbox(
                    "Novo estado",
                    STATUS_EXAME
                )

                qualidade = st.selectbox(
                    "Qualidade",
                    [
                        "",
                        "Adequada",
                        "Aceitável",
                        "Inadequada"
                    ]
                )

                motivo = st.text_input(
                    "Motivo"
                )

                guardar = st.form_submit_button(
                    "Atualizar"
                )

                if guardar:

                    execute(
                        """
                        UPDATE exames

                        SET
                            estado = ?,
                            qualidade = ?,
                            motivo_repeticao = ?

                        WHERE id = ?
                        """,
                        (
                            estado,
                            qualidade,
                            motivo,
                            opcoes[exame]
                        )
                    )

                    st.success(
                        "Exame atualizado."
                    )

                    st.rerun()


# =========================================================
# QUALIDADE
# =========================================================

elif pagina == "Qualidade":

    st.title(
        "✅ Qualidade e Segurança"
    )

    tab1, tab2 = st.tabs(
        [
            "Auditorias",
            "Nova auditoria"
        ]
    )

    with tab1:

        auditorias = query(
            """
            SELECT *
            FROM qualidade
            ORDER BY id DESC
            """
        )

        if auditorias.empty:

            st.info(
                "Ainda não existem auditorias."
            )

        else:

            total = len(
                auditorias
            )

            conformes = auditorias[
                (
                    auditorias["identificacao_ok"] == 1
                )
                &
                (
                    auditorias["lateralidade_ok"] == 1
                )
                &
                (
                    auditorias["protocolo_ok"] == 1
                )
                &
                (
                    auditorias["qualidade_ok"] == 1
                )
            ]

            taxa = (
                len(conformes)
                /
                total
                *
                100
            )

            c1, c2, c3 = st.columns(3)

            c1.metric(
                "Auditorias",
                total
            )

            c2.metric(
                "Conformes",
                len(conformes)
            )

            c3.metric(
                "Taxa conformidade",
                f"{taxa:.1f}%"
            )

            st.dataframe(
                auditorias,
                use_container_width=True,
                hide_index=True
            )

    with tab2:

        codigos = codigos_utentes()

        if not codigos:

            st.warning(
                "Primeiro adiciona um utente."
            )

        else:

            with st.form(
                "nova_auditoria"
            ):

                utente = st.selectbox(
                    "Utente",
                    codigos
                )

                identificacao = st.checkbox(
                    "Identificação confirmada",
                    value=True
                )

                lateralidade = st.checkbox(
                    "Lateralidade confirmada",
                    value=True
                )

                protocolo = st.checkbox(
                    "Protocolo correto",
                    value=True
                )

                qualidade_ok = st.checkbox(
                    "Qualidade técnica adequada",
                    value=True
                )

                observacao = st.text_area(
                    "Observação"
                )

                guardar = st.form_submit_button(
                    "Guardar auditoria"
                )

                if guardar:

                    execute(
                        """
                        INSERT INTO qualidade
                        (
                            data,
                            utente_codigo,
                            identificacao_ok,
                            lateralidade_ok,
                            protocolo_ok,
                            qualidade_ok,
                            observacao
                        )

                        VALUES (?, ?, ?, ?, ?, ?, ?)
                        """,
                        (
                            datetime.now().strftime(
                                "%Y-%m-%d %H:%M"
                            ),
                            utente,
                            int(identificacao),
                            int(lateralidade),
                            int(protocolo),
                            int(qualidade_ok),
                            observacao,
                        )
                    )

                    st.success(
                        "Auditoria guardada."
                    )

                    st.rerun()


# =========================================================
# EQUIPAMENTOS
# =========================================================

elif pagina == "Equipamentos":

    st.title(
        "🛠️ Equipamentos"
    )

    tab1, tab2, tab3 = st.tabs(
        [
            "Estado atual",
            "Adicionar equipamento",
            "Atualizar equipamento"
        ]
    )

    with tab1:

        equipamentos = query(
            """
            SELECT *
            FROM equipamentos
            ORDER BY nome
            """
        )

        if equipamentos.empty:

            st.info(
                "Ainda não existem equipamentos."
            )

        else:

            st.dataframe(
                equipamentos,
                use_container_width=True,
                hide_index=True
            )

    with tab2:

        with st.form(
            "novo_equipamento"
        ):

            nome = st.text_input(
                "Equipamento"
            )

            localizacao = st.text_input(
                "Localização"
            )

            estado = st.selectbox(
                "Estado",
                ESTADO_EQUIPAMENTO
            )

            ultima = st.text_input(
                "Última manutenção",
                placeholder="2026-08-17"
            )

            proxima = st.text_input(
                "Próxima manutenção",
                placeholder="2027-08-17"
            )

            observacao = st.text_area(
                "Observação"
            )

            guardar = st.form_submit_button(
                "Adicionar equipamento"
            )

            if guardar:

                if not nome:

                    st.error(
                        "Introduz o nome."
                    )

                else:

                    execute(
                        """
                        INSERT INTO equipamentos
                        (
                            nome,
                            localizacao,
                            estado,
                            ultima_manutencao,
                            proxima_manutencao,
                            observacao
                        )

                        VALUES (?, ?, ?, ?, ?, ?)
                        """,
                        (
                            nome,
                            localizacao,
                            estado,
                            ultima,
                            proxima,
                            observacao
                        )
                    )

                    st.success(
                        "Equipamento adicionado."
                    )

                    st.rerun()

    with tab3:

        equipamentos = query(
            """
            SELECT
                id,
                nome,
                estado

            FROM equipamentos

            ORDER BY nome
            """
        )

        if equipamentos.empty:

            st.info(
                "Ainda não existem equipamentos."
            )

        else:

            opcoes = {}

            for row in equipamentos.itertuples():

                opcoes[
                    f"{row.nome} - {row.estado}"
                ] = row.id

            with st.form(
                "atualizar_equipamento"
            ):

                equipamento = st.selectbox(
                    "Equipamento",
                    list(opcoes.keys())
                )

                estado = st.selectbox(
                    "Novo estado",
                    ESTADO_EQUIPAMENTO
                )

                observacao = st.text_area(
                    "Observação"
                )

                guardar = st.form_submit_button(
                    "Atualizar equipamento"
                )

                if guardar:

                    execute(
                        """
                        UPDATE equipamentos

                        SET
                            estado = ?,
                            observacao = ?

                        WHERE id = ?
                        """,
                        (
                            estado,
                            observacao,
                            opcoes[equipamento]
                        )
                    )

                    st.success(
                        "Equipamento atualizado."
                    )

                    st.rerun()


# =========================================================
# MELHORIA CONTÍNUA
# =========================================================

elif pagina == "Melhoria Contínua":

    st.title(
        "📈 Melhoria Contínua"
    )

    tab1, tab2 = st.tabs(
        [
            "Plano de melhoria",
            "Nova ação"
        ]
    )

    with tab1:

        melhorias = query(
            """
            SELECT *
            FROM melhorias
            ORDER BY id DESC
            """
        )

        if melhorias.empty:

            st.info(
                "Ainda não existem ações."
            )

        else:

            st.dataframe(
                melhorias,
                use_container_width=True,
                hide_index=True
            )

    with tab2:

        with st.form(
            "nova_melhoria"
        ):

            problema = st.text_area(
                "Problema identificado"
            )

            acao = st.text_area(
                "Ação de melhoria"
            )

            responsavel = st.text_input(
                "Responsável"
            )

            prazo = st.text_input(
                "Prazo",
                placeholder="2026-12-31"
            )

            estado = st.selectbox(
                "Estado",
                ESTADO_MELHORIA
            )

            resultado = st.text_area(
                "Resultado / indicador"
            )

            guardar = st.form_submit_button(
                "Guardar ação"
            )

            if guardar:

                if not problema or not acao:

                    st.error(
                        "Preenche o problema e a ação."
                    )

                else:

                    execute(
                        """
                        INSERT INTO melhorias
                        (
                            problema,
                            acao,
                            responsavel,
                            prazo,
                            estado,
                            resultado
                        )

                        VALUES (?, ?, ?, ?, ?, ?)
                        """,
                        (
                            problema,
                            acao,
                            responsavel,
                            prazo,
                            estado,
                            resultado,
                        )
                    )

                    st.success(
                        "Ação criada."
                    )

                    st.rerun()


# =========================================================
# RODAPÉ
# =========================================================

st.divider()

st.caption(
    "OrtoManager — Gestão Operacional, Qualidade e Melhoria Contínua em Ortóptica"
)
