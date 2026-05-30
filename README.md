-- ============================================================
--  PAINEL DE AGENDAMENTOS — SUPABASE
--  Execute no SQL Editor do Supabase (em ordem)
--  https://supabase.com/dashboard → SQL Editor
-- ============================================================

-- ─────────────────────────────────────────────
-- 1. TABELA PRINCIPAL
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS public.agendamentos (
    id               BIGSERIAL PRIMARY KEY,
    tipo             TEXT NOT NULL CHECK (tipo IN ('EXAME','CONSULTA')),
    recurso          TEXT NOT NULL,
    id_solicitacao   TEXT,
    data_solicitacao TIMESTAMP,
    cns              TEXT,
    paciente         TEXT,
    idade            TEXT,
    cid              TEXT,
    agendado_para    TIMESTAMP,
    situacao         TEXT DEFAULT 'Agendada',
    criado_em        TIMESTAMP DEFAULT NOW()
);

-- ─────────────────────────────────────────────
-- 2. ÍNDICES
-- ─────────────────────────────────────────────
CREATE INDEX IF NOT EXISTS idx_ag_tipo      ON public.agendamentos (tipo);
CREATE INDEX IF NOT EXISTS idx_ag_recurso   ON public.agendamentos (recurso);
CREATE INDEX IF NOT EXISTS idx_ag_data      ON public.agendamentos (agendado_para);
CREATE INDEX IF NOT EXISTS idx_ag_paciente  ON public.agendamentos (paciente);
CREATE INDEX IF NOT EXISTS idx_ag_cid       ON public.agendamentos (cid);

-- ─────────────────────────────────────────────
-- 3. ROW LEVEL SECURITY (leitura pública)
-- ─────────────────────────────────────────────
ALTER TABLE public.agendamentos ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Leitura pública"
    ON public.agendamentos FOR SELECT
    USING (true);

-- ─────────────────────────────────────────────
-- 4. VIEWS ANALÍTICAS
-- ─────────────────────────────────────────────

-- 4a. Quantitativo por data
CREATE OR REPLACE VIEW public.v_por_data AS
SELECT
    agendado_para::DATE              AS data,
    COUNT(*)                         AS total,
    COUNT(*) FILTER (WHERE tipo = 'EXAME')    AS exames,
    COUNT(*) FILTER (WHERE tipo = 'CONSULTA') AS consultas
FROM public.agendamentos
GROUP BY agendado_para::DATE
ORDER BY data;

-- 4b. Quantitativo por recurso
CREATE OR REPLACE VIEW public.v_por_recurso AS
SELECT
    recurso,
    tipo,
    COUNT(*)           AS quantidade,
    MIN(agendado_para) AS primeiro_agendamento,
    MAX(agendado_para) AS ultimo_agendamento
FROM public.agendamentos
GROUP BY recurso, tipo
ORDER BY quantidade DESC;

-- 4c. Resumo geral
CREATE OR REPLACE VIEW public.v_resumo AS
SELECT
    COUNT(*)                                          AS total,
    COUNT(*) FILTER (WHERE tipo = 'EXAME')            AS exames,
    COUNT(*) FILTER (WHERE tipo = 'CONSULTA')         AS consultas,
    COUNT(DISTINCT recurso)                           AS recursos_distintos,
    COUNT(DISTINCT agendado_para::DATE)               AS dias_com_agendamento,
    MIN(agendado_para::DATE)                          AS primeira_data,
    MAX(agendado_para::DATE)                          AS ultima_data
FROM public.agendamentos;

-- ─────────────────────────────────────────────
-- 5. FUNÇÕES RPC (chamáveis pela API REST)
-- ─────────────────────────────────────────────

-- 5a. Busca fulltext em recurso, paciente ou CID
CREATE OR REPLACE FUNCTION public.buscar_agendamentos(
    p_q     TEXT    DEFAULT '',
    p_tipo  TEXT    DEFAULT '',
    p_data  DATE    DEFAULT NULL,
    p_limit INT     DEFAULT 50,
    p_offset INT    DEFAULT 0
)
RETURNS TABLE (
    id BIGINT, tipo TEXT, recurso TEXT, id_solicitacao TEXT,
    data_solicitacao TIMESTAMP, cns TEXT, paciente TEXT,
    idade TEXT, cid TEXT, agendado_para TIMESTAMP, situacao TEXT
)
LANGUAGE sql STABLE AS $$
    SELECT id, tipo, recurso, id_solicitacao, data_solicitacao,
           cns, paciente, idade, cid, agendado_para, situacao
    FROM public.agendamentos
    WHERE
        (p_q    = '' OR recurso  ILIKE '%' || p_q || '%'
                     OR paciente ILIKE '%' || p_q || '%'
                     OR cid      ILIKE '%' || p_q || '%')
        AND (p_tipo = '' OR tipo = p_tipo)
        AND (p_data IS NULL OR agendado_para::DATE = p_data)
    ORDER BY agendado_para
    LIMIT p_limit OFFSET p_offset;
$$;

-- 5b. Agendamentos de uma data específica
CREATE OR REPLACE FUNCTION public.agendamentos_por_data(p_data DATE)
RETURNS TABLE (
    id BIGINT, tipo TEXT, recurso TEXT, paciente TEXT,
    idade TEXT, cid TEXT, agendado_para TIMESTAMP, situacao TEXT
)
LANGUAGE sql STABLE AS $$
    SELECT id, tipo, recurso, paciente, idade, cid, agendado_para, situacao
    FROM public.agendamentos
    WHERE agendado_para::DATE = p_data
    ORDER BY agendado_para;
$$;


-- ─────────────────────────────────────────────
-- 6. DADOS
-- ─────────────────────────────────────────────

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('CONSULTA','CONSULTA EM HEPATOLOGIA - HEPATITE CRONICA B','7148619','2025-10-23 13:41:25','700503343098752','JOSE DE SOUZA CARVALHO','72 ano(s), 10 meses e 7 dia(s).','B180 - Hepatite viral crônica B com agente delta','2026-06-02 12:00:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - ABDOME TOTAL','7416596','2026-01-17 12:31:45','701207085850214','TELCO GERALDO DE OLIVEIRA','51 ano(s), 3 meses e 18 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 08:12:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - ABDOME TOTAL','7416603','2026-01-17 12:45:01','703601012493133','VIVIANE BASILIO GOMES','36 ano(s), 8 meses e 22 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 08:15:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - ABDOME TOTAL','7416604','2026-01-17 12:48:24','708406702634265','MARIZETE BARBOZA DE JESUS GOMES','55 ano(s), 6 meses e 24 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 08:18:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - ABDOME TOTAL','7416612','2026-01-17 12:58:44','700003138028602','ALINE GOMES PEREIRA','49 ano(s), 11 meses e 4 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 08:21:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - ABDOME TOTAL','7416619','2026-01-17 13:08:54','702408552349822','VERA LUCIA ROSA DO NASCIMENTO','56 ano(s), 9 meses e 15 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 08:24:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - ABDOME TOTAL','7416621','2026-01-17 13:10:50','704308547759594','MARIA LUCIA DE OLIVEIRA PEGAS','56 ano(s), 4 meses e 6 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 08:30:00','Agendada'),
  ('EXAME','BIÓPSIA DE TIREÓIDE - PAAF','7834510','2026-05-19 09:01:04','704303530675797','GEISA AREDES FRANCA','32 ano(s), 3 meses e 18 dia(s).','E07  - Transtornos da tireóide','2026-06-02 09:27:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA - PEDIATRIA','7810725','2026-05-12 16:55:05','706805261454529','MIGUEL BRANDAO RIBEIRO PALMEIRA','0 ano(s), 1 meses e 28 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-02 10:30:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez - Cirurgia Torácica (Oncologia)','7669966','2026-03-31 17:15:02','702405024476120','IARA MARCIA DE SOUSA','58 ano(s), 2 meses e 10 dia(s).','C34  - Neoplasia malígna dos bronquios e dos pulmoes','2026-06-02 09:00:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez - Cirurgia Geral (Infantil)','7754908','2026-04-27 15:53:59','706208537319661','ISAAC DOS SANTOS JANUARIO DE OLIVEIRA','4 ano(s), 5 meses e 1 dia(s).','Q18  - Outras malformaçoes congenitas da face e do pescoço','2026-06-02 10:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7616488','2026-03-17 15:44:44','700508141670255','MARCIA PAIVA DE SOUZA','66 ano(s), 10 meses e 6 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 07:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7618571','2026-03-18 10:01:35','704702708977432','ROBERTO IVAN FORTES','82 ano(s), 4 meses e 10 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 07:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7618622','2026-03-18 10:06:49','707602258037596','TERESA DE JESUS CAETANO DA SILVA','77 ano(s), 3 meses e 29 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 07:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7618701','2026-03-18 10:14:58','704102185655473','APARECIDA MARIA ROCHA BARACHO','63 ano(s), 6 meses e 14 dia(s).','Z121 - Exame especial de rastreamento de neoplasia do trato intestinal','2026-06-02 07:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7618815','2026-03-18 10:26:40','708000324719620','FRANCISCO ALBERTINO DA ROCHA CARVALHO','65 ano(s), 10 meses e 7 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 07:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7618995','2026-03-18 10:46:20','700605422303864','CARLOS ALBERTO DO NASCIMENTO PEREIRA','65 ano(s), 6 meses e 0 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 07:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7619191','2026-03-18 11:09:48','700409405479742','FLAVIA DAS DORES FERREIRA FERNANDES','50 ano(s), 7 meses e 12 dia(s).','Z121 - Exame especial de rastreamento de neoplasia do trato intestinal','2026-06-02 08:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7619226','2026-03-18 11:13:34','700209953507530','MARIA GORETH REIS LEMOS','65 ano(s), 7 meses e 0 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 08:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620261','2026-03-18 13:36:41','704800582731445','MONALISA DE SOUZA FREITAS TORRES','45 ano(s), 9 meses e 29 dia(s).','K529 - Gastroenterite e colite nao-infecciosas, nao especificadas','2026-06-02 08:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620273','2026-03-18 13:38:48','702101756939195','NILSON DE FREITAS LAGE','67 ano(s), 5 meses e 12 dia(s).','K598 - Outros transtornos funcionais especificados do intestino','2026-06-02 08:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620287','2026-03-18 13:41:50','705808405755738','TANIA MARA LASNEAU BRAMY','60 ano(s), 5 meses e 23 dia(s).','R634 - Perda de peso anormal','2026-06-02 08:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620324','2026-03-18 13:49:15','706102829066130','SONIA APARECIDA TANCREDO EUZEBIO TAMIOZZO','50 ano(s), 8 meses e 4 dia(s).','Z98  - Presenca de outros estados pós-cirurgicos','2026-06-02 08:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620332','2026-03-18 13:50:48','700002678589905','GILMAR FRANCISCO LEOTERIO','41 ano(s), 4 meses e 17 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 09:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620355','2026-03-18 13:54:20','705607449573717','REGINALDO ROSA GARCIA','56 ano(s), 2 meses e 7 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 09:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620436','2026-03-18 14:05:45','705202417612279','VALDONIR GERALDO','69 ano(s), 7 meses e 6 dia(s).','K74  - Fibrose e cirrose hepáticas','2026-06-02 09:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620487','2026-03-18 14:11:56','705009488234259','CATIA RAIMUNDA CLAUDINO','47 ano(s), 10 meses e 20 dia(s).','R194 - Alteraçao do hábito intestinal','2026-06-02 09:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620563','2026-03-18 14:20:31','704808077989942','SOLANGE DE OLIVEIRA FERNANDES DUARTE','68 ano(s), 0 meses e 1 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 09:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620592','2026-03-18 14:24:19','708105695000040','ROSIMERE MOREIRA ALVES DOS SANTOS SILVA','45 ano(s), 2 meses e 18 dia(s).','R14  - Flatulencia e afecçoes correlatas','2026-06-02 09:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620688','2026-03-18 14:35:02','702404331435730','ANA MARIA VENTURA CARDOSO','67 ano(s), 8 meses e 17 dia(s).','R194 - Alteraçao do hábito intestinal','2026-06-02 10:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620701','2026-03-18 14:36:51','704604652218925','BERENICE DEL SE CH DE REZENDE','73 ano(s), 0 meses e 17 dia(s).','R194 - Alteraçao do hábito intestinal','2026-06-02 10:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620718','2026-03-18 14:38:51','700503974546758','JOANA DARC DA SILVA DIAS NOGUEIRA','73 ano(s), 11 meses e 6 dia(s).','F50  - Transtornos da alimentaçao','2026-06-02 10:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620791','2026-03-18 14:46:19','704807517229842','LIDIANE APARECIDA DA SILVA GOMES','34 ano(s), 8 meses e 8 dia(s).','K922 - Hemorragia gastrointestinal, sem outra especificaçao','2026-06-02 10:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620811','2026-03-18 14:48:21','700602413987966','ARNALDO DE ANDRADE MAIRYNK','62 ano(s), 3 meses e 20 dia(s).','Z121 - Exame especial de rastreamento de neoplasia do trato intestinal','2026-06-02 10:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620865','2026-03-18 14:54:18','705607470957219','SONIA REGINA E SILVA','60 ano(s), 3 meses e 10 dia(s).','R19  - Outros sintomas e sinais relativos ao aparelho digestivo e ao abdome','2026-06-02 10:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620896','2026-03-18 14:58:08','709806033003498','LUIZ CONSTANTINO DOS SANTOS','62 ano(s), 10 meses e 19 dia(s).','K921 - Melena','2026-06-02 11:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7620930','2026-03-18 15:01:08','703003853700577','JURANDIR DE FATIMA PEREIRA GARCIA','72 ano(s), 2 meses e 29 dia(s).','K599 - Transtorno intestinal funcional, nao especificado','2026-06-02 11:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7621105','2026-03-18 15:22:54','706708285663520','FABIO SILVA NUNES','58 ano(s), 8 meses e 15 dia(s).','Z129 - Exame especial de rastreamento de neoplasia nao especificada','2026-06-02 11:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7621396','2026-03-18 15:56:20','704507382058215','ANDERSON RODRIGUES DE ALMEIDA','51 ano(s), 10 meses e 29 dia(s).','K591 - Diarréia funcional','2026-06-02 11:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7621423','2026-03-18 16:00:13','706303129312480','IDE MACIEL DE GUSMAO','74 ano(s), 0 meses e 25 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 11:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7621524','2026-03-18 16:12:19','704109130922077','ALEX SANTANA LINO','54 ano(s), 10 meses e 2 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-02 11:50:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566312','2026-03-04 14:36:50','702601280422446','JULIANA MORAES DA SILVA','61 ano(s), 0 meses e 22 dia(s).','R100 - Abdome agudo','2026-06-02 07:00:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566362','2026-03-04 14:40:49','700509967168153','LUCIA HELENA RODRIGUES DE SOUZA','52 ano(s), 9 meses e 10 dia(s).','K70  - Doença álcoolica do fígado','2026-06-02 07:04:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566449','2026-03-04 14:49:53','708606032151985','JURACI MACHADO PEREIRA DA ROCHA','69 ano(s), 10 meses e 25 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 07:08:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566469','2026-03-04 14:51:59','708907720753110','MARIA CELIA PEREIRA DOS SANTOS','66 ano(s), 1 meses e 22 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 07:12:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566491','2026-03-04 14:54:28','700103926495417','MONIQUE MARIELLE DO CARMO DE ALMEIDA','19 ano(s), 11 meses e 12 dia(s).','K80  - Colelitiase','2026-06-02 07:16:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566521','2026-03-04 14:57:31','702905554875478','ALINE MARCIA GONCALVES','44 ano(s), 0 meses e 27 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 07:20:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566578','2026-03-04 15:03:14','700009396476905','DEIVIDSON NEVES JOAQUIM','42 ano(s), 1 meses e 24 dia(s).','R100 - Abdome agudo','2026-06-02 07:24:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566638','2026-03-04 15:10:06','700505776642655','ELISANA MARIA DE BARROS','49 ano(s), 6 meses e 29 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 07:28:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566738','2026-03-04 15:18:00','708702197345293','GISELE CLAUDIO MOREIRA SILVA','45 ano(s), 9 meses e 13 dia(s).','R10  - Dor abdominal e pélvica','2026-06-02 07:32:00','Agendada');

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('EXAME','Ultrassonografia de Aparelho Urinário','7582188','2026-03-09 12:22:34','705004628088352','CLAUDIO FURTADO DA SILVA','66 ano(s), 0 meses e 2 dia(s).','N200 - Calculose do rim','2026-06-02 07:36:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7582247','2026-03-09 12:30:45','705804486113436','ALICE MARTINS FORMAGINI SIMPLICIO','15 ano(s), 7 meses e 12 dia(s).','N30  - Cistite','2026-06-02 07:40:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7582327','2026-03-09 12:43:06','700008659019207','ADEMILSON GOMES','66 ano(s), 7 meses e 10 dia(s).','N19  - Insuficiencia renal nao especificada','2026-06-02 07:44:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7582345','2026-03-09 12:46:22','700003220795301','REGINA DE JESUS BORGES DA SILVA','67 ano(s), 0 meses e 2 dia(s).','N399 - Transtornos nao especificados do aparelho urinário','2026-06-02 07:48:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7582363','2026-03-09 12:49:37','700407439772041','RITA DE CASSIA AVELLAR FERREIRA','71 ano(s), 5 meses e 19 dia(s).','N39  - Transtornos do trato urinário','2026-06-02 07:52:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7582409','2026-03-09 12:55:51','700909979550291','ANA MARIA DA SILVA','72 ano(s), 10 meses e 4 dia(s).','N39  - Transtornos do trato urinário','2026-06-02 07:56:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7582427','2026-03-09 13:00:21','700802954165283','EDEZIO GOMES DE MELO','78 ano(s), 10 meses e 25 dia(s).','N398 - Outros transtornos especificados do aparelho urinário','2026-06-02 08:00:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7585828','2026-03-10 09:37:26','703004809946079','MARIO JANUARIO','74 ano(s), 5 meses e 7 dia(s).','N399 - Transtornos nao especificados do aparelho urinário','2026-06-02 08:04:00','Agendada'),
  ('EXAME','Ultrassonografia de Próstata','7658583','2026-03-28 11:23:12','706400658749382','CARLOS ALBERTO DOS SANTOS','55 ano(s), 5 meses e 3 dia(s).','N40  - Hiperplasia da próstata','2026-06-02 08:08:00','Agendada'),
  ('EXAME','Ultrassonografia de Próstata','7658587','2026-03-28 11:25:21','709809034269591','CESAR ROBERTO MARCELINO DA SILVA','62 ano(s), 6 meses e 25 dia(s).','N40  - Hiperplasia da próstata','2026-06-02 08:12:00','Agendada'),
  ('EXAME','Ultrassonografia de Próstata','7658590','2026-03-28 11:27:01','700002904611805','GERALDO FERREIRA BRAGA','75 ano(s), 6 meses e 3 dia(s).','N40  - Hiperplasia da próstata','2026-06-02 08:16:00','Agendada'),
  ('EXAME','Ultrassonografia de Próstata','7658593','2026-03-28 11:28:42','705803429439432','JOSE GERALDO DA SILVA','64 ano(s), 5 meses e 27 dia(s).','N40  - Hiperplasia da próstata','2026-06-02 08:20:00','Agendada'),
  ('EXAME','Ultrassonografia de Próstata','7658597','2026-03-28 11:30:44','706800231673325','ALESSANDRO DE OLIVEIRA URIAS','40 ano(s), 10 meses e 21 dia(s).','N40  - Hiperplasia da próstata','2026-06-02 08:24:00','Agendada'),
  ('EXAME','Ultrassonografia de Próstata','7658599','2026-03-28 11:32:46','708506358959978','ELCI CONCALVES RIBEIRO','64 ano(s), 10 meses e 13 dia(s).','N40  - Hiperplasia da próstata','2026-06-02 08:28:00','Agendada'),
  ('EXAME','Ultrassonografia de Próstata','7658600','2026-03-28 11:34:28','898003486336063','ROLAN SANTANA','72 ano(s), 8 meses e 14 dia(s).','N40  - Hiperplasia da próstata','2026-06-02 08:32:00','Agendada'),
  ('EXAME','Ultrassonografia de Próstata','7679663','2026-04-04 11:30:44','700001109688304','ANDRELINO RIBEIRO','61 ano(s), 11 meses e 0 dia(s).','N40  - Hiperplasia da próstata','2026-06-02 08:36:00','Agendada'),
  ('EXAME','Ultrassonografia de Próstata','7706121','2026-04-11 10:22:08','706708536067214','JOAQUIM ANTONIO DA SILVA FILHO','63 ano(s), 1 meses e 18 dia(s).','N40  - Hiperplasia da próstata','2026-06-02 08:40:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Superior','7621261','2026-03-18 15:40:46','898005825666984','RAYANI BARACHO OLIVEIRA','19 ano(s), 0 meses e 1 dia(s).','R101 - Dor localizada no abdome superior','2026-06-02 08:44:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Superior','7630597','2026-03-20 15:09:47','701800201591575','JANETE NOBREGA MARTINS RAMOS','51 ano(s), 11 meses e 23 dia(s).','K43  - Hérnia ventral','2026-06-02 08:48:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Superior','7630617','2026-03-20 15:12:12','708200136190144','PATRICIA CARLA NAZARETH DOS ANJOS MARIA','47 ano(s), 1 meses e 14 dia(s).','R101 - Dor localizada no abdome superior','2026-06-02 08:52:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Superior','7658628','2026-03-28 11:50:36','705103801794940','MATHEUS LOPES ANDRADE','20 ano(s), 9 meses e 7 dia(s).','R101 - Dor localizada no abdome superior','2026-06-02 08:56:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez - Cirurgia Geral (Oncologia)','7821388','2026-05-14 17:10:57','700007765002109','JULIO CESAR JOAQUIM TEIXEIRA','68 ano(s), 9 meses e 19 dia(s).','C15  - Neoplasia malígna do esôfago','2026-06-02 13:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Abdômen Total','7614432','2026-03-17 11:31:04','700503382586754','SEMIRAMIS DA COSTA LIMA DUQUE','48 ano(s), 8 meses e 19 dia(s).','R104 - Outras dores abdominais e as nao especificadas','2026-06-02 11:40:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (3 segmentos)','7578424','2026-03-07 11:32:11','705608462492211','FABRICIO FERREIRA RIBEIRO','49 ano(s), 2 meses e 19 dia(s).','M546 - Dor na coluna torácica','2026-06-02 17:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7658963','2026-03-28 15:52:49','705001233942358','VANESSA PEREIRA DO NASCIMENTO','48 ano(s), 6 meses e 23 dia(s).','M545 - Dor lombar baixa','2026-06-02 20:20:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7658969','2026-03-28 15:54:56','702005305191182','LEILA PEREIRA TEIXEIRA','42 ano(s), 6 meses e 4 dia(s).','M545 - Dor lombar baixa','2026-06-02 20:38:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761290','2026-04-28 15:58:11','704103893057850','ANDREA CRISTINA DA SILVA NOGUEIRA','55 ano(s), 6 meses e 24 dia(s).','H57  - Transtornos do olho e anexos','2026-06-02 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761301','2026-04-28 15:59:02','701801279480578','IZABEL BARBOSA MAXIMO','79 ano(s), 6 meses e 22 dia(s).','H57  - Transtornos do olho e anexos','2026-06-02 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761410','2026-04-28 16:12:11','700002799439505','VITOR DA SILVA ALMEIDA','48 ano(s), 11 meses e 23 dia(s).','H57  - Transtornos do olho e anexos','2026-06-02 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761446','2026-04-28 16:16:18','704000374106964','ANTONIO ALBANO DA SILVA','84 ano(s), 4 meses e 7 dia(s).','H57  - Transtornos do olho e anexos','2026-06-02 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761478','2026-04-28 16:20:10','700409424589345','ELIANA MOREIRA VICTORINO','62 ano(s), 5 meses e 14 dia(s).','H57  - Transtornos do olho e anexos','2026-06-02 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761205','2026-04-28 15:50:57','702107721556090','ISMAIL DE OLIVEIRA BORGES','55 ano(s), 5 meses e 2 dia(s).','H57  - Transtornos do olho e anexos','2026-06-03 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7719257','2026-04-15 10:04:22','706708752422620','PAULO CESAR DE ALCANTARA','58 ano(s), 5 meses e 8 dia(s).','H57  - Transtornos do olho e anexos','2026-06-03 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7653272','2026-03-26 17:20:55','700404984246643','KARINA APARECIDA VIANNA DA SILVA','21 ano(s), 11 meses e 8 dia(s).','S02  - Fratura do cranio e dos ossos da face','2026-06-03 12:50:00','Agendada'),
  ('CONSULTA','CONSULTA EM ENDOCRINOLOGIA - PEDIATRICA','7329877','2025-12-15 15:29:32','707001822707538','BERNARDO MORAES SILVA FIGUEIRA','3 ano(s), 7 meses e 18 dia(s).','E343 - Nanismo, nao classificado em outra parte','2026-06-03 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM NEUROLOGIA','7799409','2026-05-09 11:32:07','700007235788504','JULIA ROBERTA DOS SANTOS DE OLIVEIRA','29 ano(s), 4 meses e 15 dia(s).','R51  - Cefaléia','2026-06-03 08:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM ODONTOLOGIA - ODONTOPEDIATRIA','7719493','2026-04-15 10:26:36','702601726683341','ARTHUR DE AMORIM SILVA FERREIRA','10 ano(s), 4 meses e 29 dia(s).','Z008 - Outros exames gerais','2026-06-03 11:30:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez em Ortopedia - Mão (Adulto)','7755276','2026-04-27 16:37:23','702903548893077','NILMA DA SILVA CASTRO','68 ano(s), 9 meses e 18 dia(s).','S933 - Luxaçao de outras partes e das nao especificadas do pé','2026-06-03 10:30:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7566871','2026-03-04 15:31:07','703609095574532','CERINEO FERREIRA LEMOS','69 ano(s), 7 meses e 28 dia(s).','K76  - Doenças do fígado','2026-06-03 07:00:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7567042','2026-03-04 15:50:06','702108750769197','GIZEA ITAMARA ALVES PIRES','33 ano(s), 9 meses e 0 dia(s).','R101 - Dor localizada no abdome superior','2026-06-03 07:04:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7567170','2026-03-04 16:04:24','704608193199423','SUELI COSTA DE CASTRO','76 ano(s), 0 meses e 29 dia(s).','N28  - Transtornos do rim e do ureter nao classificado em outra parte','2026-06-03 07:12:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7567190','2026-03-04 16:06:36','705202417612279','VALDONIR GERALDO','69 ano(s), 7 meses e 6 dia(s).','R10  - Dor abdominal e pélvica','2026-06-03 07:16:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7754995','2026-04-27 16:02:38','707603209877390','JOSE MARCIO DA SILVA AGOSTINHO','50 ano(s), 7 meses e 14 dia(s).','M23  - Transtornos internos dos joelhos','2026-06-03 07:24:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7789903','2026-05-07 09:13:18','708009315375920','OSVANI ROX ZACARIAS DE OLIVEIRA','59 ano(s), 4 meses e 17 dia(s).','S90  - Traumatismo superficial do tornozelo e do pé','2026-06-03 07:32:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7592432','2026-03-11 12:34:33','708007399431426','PEDRO HENRIQUE VIANA ERMIDA','20 ano(s), 9 meses e 18 dia(s).','G560 - Síndrome do túnel do carpo','2026-06-03 07:40:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7592470','2026-03-11 12:40:39','706706556687518','MARIA JOSE','78 ano(s), 7 meses e 26 dia(s).','S609 - Traumatismo superficial nao especificado do punho e da mao','2026-06-03 07:44:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7592583','2026-03-11 13:04:56','700005220137004','SERGIO LUIZ SEMIAO','58 ano(s), 3 meses e 13 dia(s).','M75  - Lesoes do ombro','2026-06-03 07:48:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7596683','2026-03-12 10:49:19','700008200666209','YASMIN MACEDO DE SOUZA SANTIAGO','18 ano(s), 0 meses e 24 dia(s).','M796 - Dor em membro','2026-06-03 07:52:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7596746','2026-03-12 10:54:19','708403231141369','ROBERTO SOUZA FERNANDES','81 ano(s), 3 meses e 26 dia(s).','M796 - Dor em membro','2026-06-03 07:56:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7596830','2026-03-12 11:02:15','700004175021501','INDIARA DE SOUZA OLIVFEIRA','50 ano(s), 7 meses e 19 dia(s).','G560 - Síndrome do túnel do carpo','2026-06-03 08:00:00','Agendada');

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7596876','2026-03-12 11:06:37','709002892427612','LUCIA HELENA DA COSTA GARCIA','65 ano(s), 11 meses e 16 dia(s).','M759 - Lesao nao especificada do ombro','2026-06-03 08:04:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7597008','2026-03-12 11:19:02','702004843590283','APARECIDA DOS SANTOS CUNHA','67 ano(s), 10 meses e 8 dia(s).','M751 - Síndrome do manguito rotador','2026-06-03 08:08:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7597497','2026-03-12 12:10:17','706409602929480','TERESINHA MARIA DA CONCEICAO','72 ano(s), 1 meses e 25 dia(s).','M755 - Bursite do ombro','2026-06-03 08:12:00','Agendada'),
  ('EXAME','Ultrassonogradia de Partes Moles','7679675','2026-04-04 11:36:29','705008624093452','DIRCINEIA MARIANO COELHO','63 ano(s), 10 meses e 28 dia(s).','R229 - Tumefaçao, massa ou tumoraçao nao especificadas, localizadas','2026-06-03 08:24:00','Agendada'),
  ('EXAME','Ultrassonogradia de Partes Moles','7679694','2026-04-04 11:45:47','898001002211192','DANIELI XAVIER','46 ano(s), 3 meses e 25 dia(s).','R229 - Tumefaçao, massa ou tumoraçao nao especificadas, localizadas','2026-06-03 08:28:00','Agendada'),
  ('EXAME','Ultrassonogradia de Partes Moles','7679679','2026-04-04 11:38:22','700002879671307','ANTONIO MARCELO DOURADO MUNIZ','64 ano(s), 0 meses e 19 dia(s).','R229 - Tumefaçao, massa ou tumoraçao nao especificadas, localizadas','2026-06-03 08:32:00','Agendada'),
  ('EXAME','Ultrassonogradia de Partes Moles','7706165','2026-04-11 11:05:16','702000384948589','VIVIANE DA SILVA GAMA','44 ano(s), 0 meses e 9 dia(s).','E07  - Transtornos da tireóide','2026-06-03 08:36:00','Agendada'),
  ('EXAME','Ecocolor Dopller venoso de membro inferior (direito ou esquerdo)','7760806','2026-04-28 15:14:29','701805224661078','MARIZE ISABEL DA CUNHA SILVA','55 ano(s), 3 meses e 21 dia(s).','I87  - Transtornos das veias','2026-06-03 13:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7577380','2026-03-06 16:33:15','701200009969218','MICHELLE VIEIRA','49 ano(s), 2 meses e 21 dia(s).','F440 - Amnésia dissociativa','2026-06-03 20:20:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7577405','2026-03-06 16:38:05','708609087718785','PATRICIA DEJANIRA GERALDA DE OLIVEIRA','55 ano(s), 10 meses e 22 dia(s).','R51  - Cefaléia','2026-06-03 20:53:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7578144','2026-03-07 09:31:52','705007021967352','REGINA SILVA DE ASSIS','76 ano(s), 2 meses e 13 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-03 21:26:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7578150','2026-03-07 09:34:24','898001015095064','RAFAEL RODRIGUES DA SILVA','80 ano(s), 0 meses e 7 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-03 21:59:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7578300','2026-03-07 10:41:20','700706996832673','LUANA CRISTIE BASILIO IZAU','54 ano(s), 9 meses e 26 dia(s).','E236 - Outros transtornos da hipófise','2026-06-03 22:32:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7578609','2026-03-07 12:22:13','700004082666901','CELI REGINA DE VASCONCELLOS GAMA','72 ano(s), 6 meses e 22 dia(s).','G30  - Doença de Alzheimer','2026-06-03 23:38:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez - Planejamento em Radioterapia','7864155','2026-05-26 11:15:37','702805115550060','ANA CLAUDIA DE CARVALHO','47 ano(s), 4 meses e 14 dia(s).','C50  - Neoplasia malígna da mama','2026-06-03 09:05:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761487','2026-04-28 16:21:27','703406259424715','ALMIR MEDEIROS','52 ano(s), 8 meses e 6 dia(s).','H57  - Transtornos do olho e anexos','2026-06-03 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761495','2026-04-28 16:22:15','702003824623782','LENICE BASILIO DA SILVA','72 ano(s), 7 meses e 10 dia(s).','H57  - Transtornos do olho e anexos','2026-06-03 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761508','2026-04-28 16:23:29','708000392686228','LUZIA PEDRINA DOS SANTOS COSTA','82 ano(s), 10 meses e 20 dia(s).','H57  - Transtornos do olho e anexos','2026-06-03 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7761518','2026-04-28 16:25:12','704000349404663','ASSIS MIGUEL DA SILVA','70 ano(s), 1 meses e 3 dia(s).','H57  - Transtornos do olho e anexos','2026-06-03 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7764007','2026-04-29 10:53:17','708009328904226','MARIA DE LOURDES MELO','68 ano(s), 0 meses e 11 dia(s).','H57  - Transtornos do olho e anexos','2026-06-03 07:00:00','Agendada'),
  ('EXAME','Cateterismo Cardíaco (Internados)','7861532','2026-05-25 16:50:20','707808659725918','JOSE RAMON GALLUCCI RODRIGUEZ','63 ano(s), 7 meses e 5 dia(s).','I200 - Angina instável','2026-06-03 10:00:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7838528','2026-05-19 15:50:13','701002821055393','ROSA MARIA DA SILVEIRA RODRIGUES COSTA','77 ano(s), 11 meses e 29 dia(s).','M859 - Transtorno nao especificado da densidade e da estrutura ósseas','2026-06-04 08:59:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (3 segmentos)','7587388','2026-03-10 12:28:52','702403058426620','MARINES DOS SANTOS CALHEIRO SOUZA','61 ano(s), 2 meses e 25 dia(s).','M51  - Transtornos de discos intervertebrais','2026-06-04 17:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (2 segmentos)','7679900','2026-04-04 14:53:29','704109130922077','ALEX SANTANA LINO','54 ano(s), 10 meses e 2 dia(s).','M51  - Transtornos de discos intervertebrais','2026-06-05 14:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (2 segmentos)','7712372','2026-04-13 17:38:48','702403096402520','ILCEMERY DO CARMO DUTRA','60 ano(s), 9 meses e 27 dia(s).','M542 - Cervicalgia','2026-06-05 15:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7669348','2026-03-31 15:42:46','705201495346475','HENDRIX LUIZ PEREIRA RODRIGUES','41 ano(s), 8 meses e 20 dia(s).','M545 - Dor lombar baixa','2026-06-05 22:08:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7670087','2026-03-31 17:57:35','700007368990908','JANE MACHADO FERREIRA GAMA','65 ano(s), 10 meses e 13 dia(s).','M545 - Dor lombar baixa','2026-06-05 22:26:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7675094','2026-04-01 16:49:27','708105516665530','SILVIANE SANTOS DA SILVA','43 ano(s), 4 meses e 3 dia(s).','M51  - Transtornos de discos intervertebrais','2026-06-05 22:44:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7679895','2026-04-04 14:48:58','700400452253846','THAIS ALMEIDA DE OLIVEIRA','26 ano(s), 2 meses e 24 dia(s).','M545 - Dor lombar baixa','2026-06-05 23:02:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7679909','2026-04-04 15:04:46','703208680031892','LEANDRO ALVES DE ALMEIDA AROUCA','43 ano(s), 10 meses e 4 dia(s).','M51  - Transtornos de discos intervertebrais','2026-06-05 23:20:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7679921','2026-04-04 15:15:33','700501565313658','WANDERLEA DOS SANTOS ARAUJO','34 ano(s), 6 meses e 28 dia(s).','M545 - Dor lombar baixa','2026-06-05 23:38:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7679930','2026-04-04 15:24:03','702306127748710','CLAUDIA REGINA DE MENEZES ASSIS','60 ano(s), 6 meses e 26 dia(s).','M545 - Dor lombar baixa','2026-06-05 23:56:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7764024','2026-04-29 10:54:48','700005862693305','PATRICIA CANDIDO DE AGUIAR DE VASCONCELOS','51 ano(s), 6 meses e 4 dia(s).','H57  - Transtornos do olho e anexos','2026-06-05 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7764270','2026-04-29 11:20:25','706000873401844','MIRIAN ROSA RAMOS DA SILVA','50 ano(s), 0 meses e 2 dia(s).','H57  - Transtornos do olho e anexos','2026-06-05 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7764356','2026-04-29 11:29:06','702402064008427','MARLI LOPES DE ALMEIDA','76 ano(s), 9 meses e 0 dia(s).','H57  - Transtornos do olho e anexos','2026-06-05 07:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7764415','2026-04-29 11:35:07','705002644698958','JANAINA SILVA DE SOUSA','42 ano(s), 10 meses e 20 dia(s).','H57  - Transtornos do olho e anexos','2026-06-05 07:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7679934','2026-04-04 15:28:42','703403237110014','ROSIMERI GONCALVES SANTOS SILVA','59 ano(s), 8 meses e 29 dia(s).','M545 - Dor lombar baixa','2026-06-06 18:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Coluna (1 Segmento)','7679949','2026-04-04 15:38:20','898003912245241','GILSON RIBEIRO DA SILVA','55 ano(s), 11 meses e 19 dia(s).','M545 - Dor lombar baixa','2026-06-06 18:48:00','Agendada'),
  ('EXAME','Ressonância Magnética do Membro Superior Esquerdo','7578534','2026-03-07 11:58:41','707009847033732','LUCIANA SOARES BARROSO','57 ano(s), 0 meses e 3 dia(s).','M75  - Lesoes do ombro','2026-06-07 16:30:00','Agendada'),
  ('EXAME','Ressonância Magnética do Membro Superior Esquerdo','7585757','2026-03-10 09:29:59','706903125678434','BIANCA ANASTACIA BENEDITO DA SILVA','38 ano(s), 1 meses e 7 dia(s).','S60  - Traumatismo superficial do punho e da mao','2026-06-07 17:06:00','Agendada'),
  ('EXAME','Ressonância Magnética do Membro Superior Esquerdo','7585975','2026-03-10 09:52:49','700502949007457','HALF BARBOSA','37 ano(s), 11 meses e 26 dia(s).','M75  - Lesoes do ombro','2026-06-07 17:24:00','Agendada'),
  ('EXAME','Ressonância Magnética do Membro Superior Esquerdo','7586437','2026-03-10 10:53:15','708003340147329','FABIANA DA SILVA CHRISOSTIMO','45 ano(s), 6 meses e 9 dia(s).','M75  - Lesoes do ombro','2026-06-07 17:42:00','Agendada'),
  ('EXAME','Ressonância Magnética do Membro Superior Esquerdo','7594533','2026-03-11 17:35:28','704201781166086','MAURECI SILVA DOS SANTOS','55 ano(s), 7 meses e 9 dia(s).','M755 - Bursite do ombro','2026-06-07 18:36:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7578401','2026-03-07 11:23:25','700700916939772','CLAUDIANE DE OLIVEIRA SILVA','53 ano(s), 5 meses e 18 dia(s).','E221 - Hiperprolactinemia','2026-06-07 07:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7578614','2026-03-07 12:23:25','708209615040442','PAULO DE OLIVEIRA LOPES','74 ano(s), 3 meses e 20 dia(s).','I69  - Doenças cerebrovasculares','2026-06-07 08:05:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7578804','2026-03-07 13:39:36','700001918576305','ANA PAULA APARECIDA FIRMINO','42 ano(s), 8 meses e 18 dia(s).','I67  - Doenças cerebrovasculares','2026-06-07 08:40:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7580483','2026-03-09 08:52:51','708500569806580','MARCIA MARIA DE SOUZA','60 ano(s), 8 meses e 19 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-07 09:15:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7580452','2026-03-09 08:47:25','702009377964084','JUSSARA OLIVEIRA TEIXEIRA','75 ano(s), 2 meses e 4 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-07 09:50:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7580690','2026-03-09 09:23:46','700005230822001','LUIS CARLOS DA SILVA','62 ano(s), 4 meses e 10 dia(s).','G93  - Transtornos do encefalo','2026-06-07 10:25:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7581487','2026-03-09 11:02:10','705001051978756','ENEDINA DE OLIVEIRA BORGES','50 ano(s), 5 meses e 24 dia(s).','R51  - Cefaléia','2026-06-07 11:35:00','Agendada');

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('EXAME','Ressonância Magnética de Crânio','7582926','2026-03-09 14:18:53','702006302150087','MARIA HELENA DA SILVA','72 ano(s), 5 meses e 21 dia(s).','G46  - Síndromes vasculares cerebrais que ocorrem em doenças cerebrovasculares','2026-06-07 12:10:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7730973','2026-04-17 11:49:24','705003265588152','ANGELA MARIA FIRMINO DA COSTA BRAGA','73 ano(s), 2 meses e 20 dia(s).','H57  - Transtornos do olho e anexos','2026-06-08 13:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7739650','2026-04-20 16:41:02','706709531156317','JOAO CARLOS MARIANO DA SILVA','69 ano(s), 9 meses e 6 dia(s).','H57  - Transtornos do olho e anexos','2026-06-08 13:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7759779','2026-04-28 13:13:32','700104974072017','JOSIANA LEMOS BARBOSA','46 ano(s), 2 meses e 9 dia(s).','H57  - Transtornos do olho e anexos','2026-06-08 13:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7759788','2026-04-28 13:15:27','898004150386295','MARIA INES CARDOZO DE SA VILA VERDE','66 ano(s), 6 meses e 1 dia(s).','H57  - Transtornos do olho e anexos','2026-06-08 13:00:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7846938','2026-05-21 11:38:43','709608670818577','RUBIA DINIZ DE AVELLAR','55 ano(s), 9 meses e 0 dia(s).','M818 - Outras osteoporoses','2026-06-08 07:51:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7846958','2026-05-21 11:40:49','702806695559469','ROBERIA LUCIA RIBEIRO TEIXEIRA VIVIANI','64 ano(s), 6 meses e 5 dia(s).','M818 - Outras osteoporoses','2026-06-08 08:08:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7846981','2026-05-21 11:43:02','700900903610295','LUCINEIA DOS SANTOS VIVEIROS ERNESTO','47 ano(s), 9 meses e 8 dia(s).','M818 - Outras osteoporoses','2026-06-08 08:25:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847013','2026-05-21 11:46:05','704806081859347','MARIA JOSE DE FREITAS CAMPBELL','67 ano(s), 2 meses e 15 dia(s).','M818 - Outras osteoporoses','2026-06-08 08:42:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847045','2026-05-21 11:49:42','707403013603370','MARIA ELIANE DE OLIVEIRA LEMOS DOS SANTOS','60 ano(s), 2 meses e 5 dia(s).','M818 - Outras osteoporoses','2026-06-08 08:59:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847069','2026-05-21 11:52:19','706007835294641','VALERIA RIBEIRO DE OLIVEIRA','60 ano(s), 4 meses e 17 dia(s).','M818 - Outras osteoporoses','2026-06-08 09:16:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847084','2026-05-21 11:54:18','704207222904080','JESSICA VELOSO PEROZINI','32 ano(s), 8 meses e 24 dia(s).','M818 - Outras osteoporoses','2026-06-08 09:33:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847171','2026-05-21 12:04:19','706702525915817','MONICA FONTES MELO NOBREGA','48 ano(s), 2 meses e 1 dia(s).','M810 - Osteoporose pós-menopáusica','2026-06-08 10:07:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847197','2026-05-21 12:07:52','704108189309877','NELY TEIXEIRA','80 ano(s), 5 meses e 8 dia(s).','M818 - Outras osteoporoses','2026-06-08 10:24:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847213','2026-05-21 12:09:31','702106716119396','MEIRY LUCIA SANTOS OLIVEIRA','55 ano(s), 1 meses e 15 dia(s).','M818 - Outras osteoporoses','2026-06-08 10:41:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847226','2026-05-21 12:11:17','705007289063355','ELIZABETE FERREIRA','50 ano(s), 1 meses e 15 dia(s).','M818 - Outras osteoporoses','2026-06-08 10:58:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7851916','2026-05-22 11:02:01','707601233605999','MARIA APARECIDA DA CONCEICAO MOREIRA','68 ano(s), 9 meses e 28 dia(s).','M818 - Outras osteoporoses','2026-06-08 13:48:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA','7646544','2026-03-25 12:48:58','700607403681362','ANDREA DOS SANTOS SILVA','55 ano(s), 2 meses e 2 dia(s).','I51  - Doenças cardíacas maldefinidas','2026-06-08 08:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA','7646967','2026-03-25 14:03:02','700308965402133','MILTON FRANCISCO','57 ano(s), 7 meses e 6 dia(s).','I51  - Doenças cardíacas maldefinidas','2026-06-08 08:20:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA','7646996','2026-03-25 14:06:46','700408356957150','FABIO FIGUEIRA DOS SANTOS','45 ano(s), 0 meses e 1 dia(s).','I51  - Doenças cardíacas maldefinidas','2026-06-08 08:40:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA','7647203','2026-03-25 14:31:58','708206632036242','LEANDRA DA SILVA PEREIRA','43 ano(s), 10 meses e 25 dia(s).','I51  - Doenças cardíacas maldefinidas','2026-06-08 09:00:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7658696','2026-03-28 12:37:01','206150576100006','ROOSEVELT CAMPOS','83 ano(s), 6 meses e 29 dia(s).','I83  - Varizes dos membros inferiores','2026-06-08 13:00:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7658707','2026-03-28 12:41:08','704602652196229','ELISANGELA OLIVEIRA COSTA','46 ano(s), 9 meses e 7 dia(s).','I83  - Varizes dos membros inferiores','2026-06-08 13:06:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7658711','2026-03-28 12:42:55','706500353759391','CRISTINA DA SILVA ESCRIVANI','61 ano(s), 10 meses e 10 dia(s).','I83  - Varizes dos membros inferiores','2026-06-08 13:12:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7658701','2026-03-28 12:38:45','703509023052530','ANTONIO CARLOS GRIJO COELHO','62 ano(s), 9 meses e 0 dia(s).','I83  - Varizes dos membros inferiores','2026-06-08 13:00:00','Agendada'),
  ('EXAME','TOMOGRAFIA CONE BEAN','7864770','2026-05-26 12:19:43','700008052783202','VITORIA RODRIGUES SILVERIO','28 ano(s), 7 meses e 28 dia(s).','K011 - Dentes impactados','2026-06-08 11:00:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7624879','2026-03-19 13:35:22','700400996955241','SARA DE SOUZA SANTOS DA SILVA','25 ano(s), 6 meses e 22 dia(s).','R10  - Dor abdominal e pélvica','2026-06-08 08:48:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7624892','2026-03-19 13:37:35','704809079398144','DULCINEA BENEDITO SANTOS SILVA','83 ano(s), 1 meses e 8 dia(s).','R10  - Dor abdominal e pélvica','2026-06-08 08:57:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7624944','2026-03-19 13:45:23','703403157330700','MARIA EDUARDA RODRIGUES NOGUEIRA','22 ano(s), 0 meses e 26 dia(s).','K29  - Gastrite e duodenite','2026-06-08 09:06:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7624963','2026-03-19 13:47:35','703603079309131','RICARDO DE SOUZA','50 ano(s), 7 meses e 14 dia(s).','R10  - Dor abdominal e pélvica','2026-06-08 09:15:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7625007','2026-03-19 13:52:32','898003453207639','CLAUDIONIRA DA SILVA CORREIA','55 ano(s), 2 meses e 21 dia(s).','K29  - Gastrite e duodenite','2026-06-08 09:24:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7625043','2026-03-19 13:56:49','898003951576903','CLEIDE SUELI PEREIRA MACHADO','59 ano(s), 9 meses e 6 dia(s).','K21  - Doença de refluxo gastroesofágico','2026-06-08 09:33:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7628961','2026-03-20 11:32:51','705602494224616','ROSELI BATISTA DE SOUZA CARVALHO','59 ano(s), 10 meses e 25 dia(s).','K30  - DISPEPSIA','2026-06-08 10:00:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7629000','2026-03-20 11:36:47','700509573752751','MELISSA DA CUNHA SILVA DOS SANTOS','23 ano(s), 4 meses e 28 dia(s).','K296 - Outras gastrites','2026-06-08 10:09:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7629307','2026-03-20 12:11:02','700601460666460','PAULO SERGIO EVARISTO FERREIRA','63 ano(s), 7 meses e 24 dia(s).','K29  - Gastrite e duodenite','2026-06-08 10:18:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7629321','2026-03-20 12:13:27','708100576303037','ROSELI DA CUNHA BONIFACIO DOS SANTOS','66 ano(s), 6 meses e 20 dia(s).','K290 - Gastrite hemorrágica aguda','2026-06-08 10:27:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7629411','2026-03-20 12:25:11','706507316180195','RUAN DIAS DE ALMEIDA','34 ano(s), 2 meses e 22 dia(s).','K29  - Gastrite e duodenite','2026-06-08 10:45:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7629431','2026-03-20 12:27:54','704001361558668','MARILIA SILVEIRA TEIXEIRA','65 ano(s), 11 meses e 23 dia(s).','K922 - Hemorragia gastrointestinal, sem outra especificaçao','2026-06-08 10:54:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7629446','2026-03-20 12:29:47','706002368842644','MARIA APARECIDA ANTONIO','70 ano(s), 3 meses e 20 dia(s).','R101 - Dor localizada no abdome superior','2026-06-08 11:12:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7629876','2026-03-20 13:47:04','708006834732020','JULIANA SALES CAMPOS DA SILVA','36 ano(s), 10 meses e 6 dia(s).','K29  - Gastrite e duodenite','2026-06-08 11:21:00','Agendada'),
  ('EXAME','Endoscopia com biópsia','7629936','2026-03-20 13:57:28','707107809520920','CLAUDINEIA APARECIDA OLIVEIRA DA SILVA','32 ano(s), 11 meses e 23 dia(s).','K296 - Outras gastrites','2026-06-08 11:30:00','Agendada'),
  ('EXAME','Eco Carotidas e Vertebrais - com e sem Doppler','7877576','2026-05-28 15:32:49','704805548952242','SEBASTIAO BARBOZA DOS SANTOS','70 ano(s), 7 meses e 27 dia(s).','I87  - Transtornos das veias','2026-06-08 07:45:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7677699','2026-04-02 14:49:33','708407732393062','MARIA DE FATIMA DUTRA LOPES','70 ano(s), 2 meses e 21 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-08 08:15:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7677775','2026-04-02 15:09:28','703606026156333','TAMIRES VIDAL DE PAULA','31 ano(s), 11 meses e 16 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-08 08:20:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7677785','2026-04-02 15:12:25','708603564749084','SUELLEN DOS SANTOS COSTA DA SILVA','28 ano(s), 10 meses e 14 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-08 08:25:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7677859','2026-04-02 15:29:37','706102842715430','ALESSANDRO ESTEVAO DE MOURA','52 ano(s), 8 meses e 1 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-08 08:30:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7694937','2026-04-08 14:28:42','703401246072918','EDIR PEREIRA DA COSTA','74 ano(s), 9 meses e 8 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 08:35:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7697179','2026-04-09 08:44:26','709203292395530','PAOLA BORGES JUSTINO','32 ano(s), 7 meses e 28 dia(s).','Z13  - Exame especial de rastreamento (screening) de outros transtornos e doenças','2026-06-08 08:40:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7697858','2026-04-09 10:15:10','702006818015280','APARECIDA DE JESUS SANTOS VARGAS','43 ano(s), 5 meses e 29 dia(s).','Z13  - Exame especial de rastreamento (screening) de outros transtornos e doenças','2026-06-08 08:45:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7706340','2026-04-11 13:46:49','700009797754604','SANDRA REGINA DE DEUS PIASSA','70 ano(s), 3 meses e 27 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-08 08:50:00','Agendada');

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7709524','2026-04-13 11:31:36','706407600874687','CRISTIANE DOS SANTOS E SANTOS','47 ano(s), 1 meses e 13 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 08:55:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7709599','2026-04-13 11:38:24','709802086733799','ANA MARIA TEODORO','63 ano(s), 6 meses e 13 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:00:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7709644','2026-04-13 11:42:24','700008322031704','ENILCE DE LIMA PORTO','66 ano(s), 1 meses e 1 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:05:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7709944','2026-04-13 12:19:25','708702174712397','NILZA DE LIMA DIAS','80 ano(s), 9 meses e 17 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:10:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7709962','2026-04-13 12:22:12','706805750377721','ANGELA NEVES DE MATTOS','62 ano(s), 9 meses e 28 dia(s).','Z135 - Exame especial de rastreamento de doenças dos ouvidos e dos olhos','2026-06-08 09:15:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7710002','2026-04-13 12:26:57','708001856954528','JAIR GONCALVES DANTAS','73 ano(s), 8 meses e 13 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:20:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7710027','2026-04-13 12:30:23','700002084192905','SANDRA MARIA DOS ANJOS GOMES','49 ano(s), 4 meses e 7 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:25:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7710074','2026-04-13 12:38:57','701404629534732','MARLENE DOS SANTOS FURTADO','53 ano(s), 3 meses e 10 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:30:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7710105','2026-04-13 12:44:07','702804616066667','MARIA APARECIDA ALVES','74 ano(s), 2 meses e 20 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:35:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7710114','2026-04-13 12:46:38','705209416420879','ISABEL MARIA CESARIO','70 ano(s), 2 meses e 1 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:40:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7710145','2026-04-13 12:51:46','705400439598291','DILSON DE OLIVEIRA','61 ano(s), 2 meses e 26 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:45:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7710164','2026-04-13 12:54:54','704602122624622','ALDAIR CORREA DO ESPIRITO SANTO','62 ano(s), 8 meses e 19 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:50:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7710176','2026-04-13 12:57:21','700000617864505','EDELUCIA DA SILVA','50 ano(s), 6 meses e 2 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 09:55:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7711718','2026-04-13 15:52:05','702303056961120','TEREZA ANA OLIVEIRA SILVA','70 ano(s), 7 meses e 14 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:00:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7713576','2026-04-14 09:27:05','705005002628155','LUCIANA DOS SANTOS RODRIGUES APOLINARIO','54 ano(s), 6 meses e 2 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:05:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7713935','2026-04-14 10:01:00','708401258746561','ATAIDE GOMES DA SILVA','60 ano(s), 3 meses e 7 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:15:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7713998','2026-04-14 10:06:54','702401530401027','CARLOS REINALDO DA SILVA','72 ano(s), 0 meses e 11 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:20:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7714075','2026-04-14 10:14:37','702001836445882','JULIANA DE OLIVEIRA','41 ano(s), 10 meses e 19 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:25:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7714101','2026-04-14 10:17:23','700403424084345','ADRIANA DE CARVALHO BALBINO','46 ano(s), 5 meses e 19 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:30:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7714148','2026-04-14 10:21:41','702000367765781','JOAO CARLOS ABGAEL DOS REIS','35 ano(s), 2 meses e 27 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:35:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7714215','2026-04-14 10:28:23','705008609567953','HERMANO JOSE DA SILVA','54 ano(s), 1 meses e 23 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:40:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7714261','2026-04-14 10:31:49','708706196768496','JADERSON TEODOZIO DOS SANTOS','24 ano(s), 3 meses e 2 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:45:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7714305','2026-04-14 10:35:04','708606021554285','IRENE MARIA SOARES GALDINO','77 ano(s), 2 meses e 20 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:50:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715073','2026-04-14 11:45:57','706207052004862','IVETE LINHARES DE SOUZA','75 ano(s), 0 meses e 24 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 10:55:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715191','2026-04-14 11:58:49','702106789648194','ELZA SUELI LOPES DA ROCHA','74 ano(s), 0 meses e 4 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:00:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715212','2026-04-14 12:00:55','706208033867460','CRISTINA DE FATIMA CARDOSO','68 ano(s), 6 meses e 1 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:05:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715286','2026-04-14 12:09:50','708507337844471','MARIA LUIZA DOS REIS AMARAL','78 ano(s), 11 meses e 20 dia(s).','Z136 - Exame especial de rastreamento de doenças cardiovasculares','2026-06-08 11:10:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715319','2026-04-14 12:14:59','702601702568542','MARLI SEBASTIAO DE SOUZA','78 ano(s), 6 meses e 8 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:15:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715331','2026-04-14 12:17:53','706501361859899','MARIA HELENA DA CRUZ','49 ano(s), 0 meses e 22 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:20:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715344','2026-04-14 12:21:08','704609662356527','NEIDE APARECIDA ROCHA DE OLIVEIRA','55 ano(s), 0 meses e 2 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:25:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715366','2026-04-14 12:25:43','700005831330707','ALESSANDRA SOARES GALDINO','58 ano(s), 5 meses e 1 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:30:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715379','2026-04-14 12:28:56','702403523983220','ROSANGELA HELENA DE SOUZA','47 ano(s), 5 meses e 10 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:35:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715503','2026-04-14 12:53:03','707102372114620','ILDA COUTO DA SILVA','77 ano(s), 11 meses e 16 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:40:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715612','2026-04-14 13:18:20','700002831384901','ELISANGELA SILVA','51 ano(s), 2 meses e 1 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:45:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7715650','2026-04-14 13:25:34','706406641515885','LUIZ CARLOS GUIMARAES DOS SANTOS','62 ano(s), 0 meses e 3 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 11:50:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7716496','2026-04-14 15:06:36','898003964028683','GISELE DUARTE EMILIO','40 ano(s), 4 meses e 5 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-08 11:55:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7716598','2026-04-14 15:16:12','700005595763609','JOZIMAR DOS SANTOS','69 ano(s), 6 meses e 27 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:00:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7716626','2026-04-14 15:19:32','706009396726846','MARCELO DA SILVA NASCIMENTO','51 ano(s), 11 meses e 10 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:05:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7716710','2026-04-14 15:28:05','702405565936826','ROGERIO MOTA DO AMARAL','66 ano(s), 7 meses e 28 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:10:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7716777','2026-04-14 15:34:21','702808624857760','DEYSE DE SOUZA ROCHA','40 ano(s), 8 meses e 23 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:15:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7718718','2026-04-15 09:03:58','704000863139369','MARTA CRISTINA DE ALMEIDA','67 ano(s), 4 meses e 0 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:20:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7718735','2026-04-15 09:06:09','703002840381779','LUIZA HELENA COELHO','61 ano(s), 4 meses e 27 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:25:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7718920','2026-04-15 09:27:26','704505331310519','MARIA DAS GRACAS LOPES RIBEIRO','76 ano(s), 11 meses e 15 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:30:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7718942','2026-04-15 09:30:12','700809429958884','MARCIA MARIA DA SILVA','57 ano(s), 0 meses e 23 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:35:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7719202','2026-04-15 09:58:33','704805551433145','DINEA BARBOSA RAPOSO','79 ano(s), 7 meses e 3 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:40:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7720214','2026-04-15 11:32:16','702400049706125','LUIZ FERNANDO DA SILVA CHAGAS DOS SANTOS','37 ano(s), 1 meses e 8 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-08 12:45:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7720877','2026-04-15 12:58:06','707002813024637','EBER SOARES DE SOUZA','71 ano(s), 11 meses e 20 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:50:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7720895','2026-04-15 13:01:34','708202165062940','MARIA JULIA DA SILVA JOVELINO','56 ano(s), 10 meses e 29 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 12:55:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7720906','2026-04-15 13:04:21','700603969734869','MARISE DE SOUZA PEREIRA','68 ano(s), 10 meses e 3 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 13:00:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7720938','2026-04-15 13:11:52','708401707259964','JOSE CARLOS DE OLIVEIRA FERREIRA','67 ano(s), 3 meses e 9 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 13:05:00','Agendada');

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7720965','2026-04-15 13:17:09','700801954206485','GIOVANNA ROCHA SOUZA','23 ano(s), 4 meses e 19 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 13:10:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7721009','2026-04-15 13:25:27','702000803882182','FELIX DOS SANTOS','60 ano(s), 4 meses e 27 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 13:15:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7721028','2026-04-15 13:28:13','700504110383552','ANA ANGELICA CISCOTTO DE ALMEIDA GUIMARAES','61 ano(s), 10 meses e 13 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 13:20:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7721855','2026-04-15 15:11:47','708403230804860','CELIO MARQUES PORTUGAL','55 ano(s), 6 meses e 10 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 13:25:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7721885','2026-04-15 15:14:27','704602652196229','ELISANGELA OLIVEIRA COSTA','46 ano(s), 9 meses e 7 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 13:30:00','Agendada'),
  ('EXAME','Ecocardiograma Transtorácico (Ambulatorial)','7722237','2026-04-15 15:48:54','700507744466454','ELISABETE GOMES MENDES','68 ano(s), 8 meses e 6 dia(s).','I50  - Insuficiencia cardíaca','2026-06-08 13:35:00','Agendada'),
  ('EXAME','Angiorressonância Cerebral','7825277','2026-05-15 14:54:26','702801507480370','VANESSA CRISTINA CELESTINO DA SILVA','29 ano(s), 9 meses e 29 dia(s).','R51  - Cefaléia','2026-06-08 07:40:00','Agendada'),
  ('EXAME','Angiorressonância Cerebral','7833465','2026-05-18 17:59:31','702803687465962','SUE ELLEN MEDEIROS MOLINARI','43 ano(s), 10 meses e 24 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-08 08:15:00','Agendada'),
  ('EXAME','Angiorressonância Cerebral','7586964','2026-03-10 11:43:28','704809570665343','GLORIA MARIA SACCHI DE OLIVEIRA','72 ano(s), 9 meses e 15 dia(s).','R42  - Tontura e instabilidade','2026-06-08 09:25:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7583903','2026-03-09 15:55:50','703401234888810','PAULO CESAR ROSAS','74 ano(s), 6 meses e 9 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-08 11:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7584456','2026-03-09 17:07:54','898006216665334','OLIVER GAEL DE OLIVEIRA CELSO','5 ano(s), 8 meses e 12 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-08 11:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7585688','2026-03-10 09:23:01','700001365554400','ENEZIANA DE OLIVEIRA CABRAL','78 ano(s), 0 meses e 22 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-08 12:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM ENDOCRINOLOGIA - PEDIATRICA','7329922','2025-12-15 15:35:38','898005112877636','JOAO MIGUEL MONTEIRO DA SILVA','10 ano(s), 7 meses e 15 dia(s).','E34  - Transtornos endocrinos','2026-06-09 13:00:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847263','2026-05-21 12:16:32','708502343093070','SEBASTIANA MARQUES CARLOS','73 ano(s), 3 meses e 9 dia(s).','M818 - Outras osteoporoses','2026-06-09 07:00:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7847294','2026-05-21 12:20:32','704102147050578','DIONISIO PINHEIRO FREDERICO','53 ano(s), 7 meses e 2 dia(s).','M85  - Transtornos da densidade e da estrutura osseas','2026-06-09 07:17:00','Agendada'),
  ('CONSULTA','CONSULTA EM ENDOCRINOLOGIA - PEDIATRICA','7506395','2026-02-11 14:23:11','702505337779733','MARCO AURELIO DO SANTOS CARVALHO','13 ano(s), 0 meses e 13 dia(s).','E66  - Obesidade','2026-06-09 12:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM ENDOCRINOLOGIA - PEDIATRICA','7351743','2025-12-22 14:25:20','702404565162323','NOAH ASSUNCAO CORREA','1 ano(s), 1 meses e 27 dia(s).','R635 - Ganho de peso anormal','2026-06-09 12:00:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez - Cirurgia Hepatobiliar (Oncologia)','7808778','2026-05-12 13:07:14','704602147539920','HERIVELTO ROSTIROLE CESAR','79 ano(s), 10 meses e 10 dia(s).','C22  - Neoplasia malígna do fígado e das vias biliares intra-hepáticas','2026-06-09 08:00:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7669503','2026-03-31 16:04:54','705404443198697','IZABELLA CABRAL BARBOSA DA SILVA','22 ano(s), 11 meses e 3 dia(s).','I87  - Transtornos das veias','2026-06-09 15:56:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7666052','2026-03-31 09:22:44','709505609626270','MARIA APARECIDA RODRIGUES DOS SANTOS','65 ano(s), 2 meses e 10 dia(s).','I739 - Doenças vasculares periféricas nao especificada','2026-06-09 09:00:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7666180','2026-03-31 09:37:51','700508933929851','JOSE CARLOS TELES DE VASCONCELOS','69 ano(s), 6 meses e 17 dia(s).','I878 - Outros transtornos venosos especificados','2026-06-09 09:08:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7666236','2026-03-31 09:44:38','703004852639573','GILLIARD ALEXANDRE DA SILVA','41 ano(s), 4 meses e 18 dia(s).','I87  - Transtornos das veias','2026-06-09 09:16:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7669672','2026-03-31 16:27:46','702106716119396','MEIRY LUCIA SANTOS OLIVEIRA','55 ano(s), 1 meses e 15 dia(s).','I87  - Transtornos das veias','2026-06-09 09:24:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7669699','2026-03-31 16:30:56','700009486582003','JENIFER OLIVEIRA DE ALMEIDA MODESTO','20 ano(s), 6 meses e 17 dia(s).','I879 - Transtorno venoso nao especificado','2026-06-09 09:32:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez - Cirurgia Geral (Infantil)','7864236','2026-05-26 11:22:59','701002889006398','SAMUEL BRANDAO FONSECA','3 ano(s), 2 meses e 9 dia(s).','N47  - Hipertrofia do prepúcio, fimose e parafimose','2026-06-09 11:30:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez - Cirurgia Geral (Infantil)','7869003','2026-05-27 10:19:32','705008295749051','OLIVER DELFINO CARVALHO','0 ano(s), 3 meses e 17 dia(s).','Q43  - Outras malformaçoes congenitas do intestino','2026-06-09 11:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7621613','2026-03-18 16:25:49','706209098092669','LUCIANA DO NASCIMENTO SODRE','47 ano(s), 10 meses e 8 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 07:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7621638','2026-03-18 16:29:09','705402444592796','GABRIELA DA SILVA','36 ano(s), 5 meses e 6 dia(s).','K590 - Constipaçao','2026-06-09 07:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7621712','2026-03-18 16:40:47','706209003338460','MARCIA SILVA ROSA DE SOUSA','62 ano(s), 7 meses e 29 dia(s).','R10  - Dor abdominal e pélvica','2026-06-09 07:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7621766','2026-03-18 16:48:34','702602717371147','ELZA PAULO','68 ano(s), 7 meses e 7 dia(s).','K57  - Doença diverticular do intestino','2026-06-09 07:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7621790','2026-03-18 16:52:20','702503365215030','ROSANGELA DOS SANTOS NEVES ALVES','65 ano(s), 5 meses e 15 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 07:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623142','2026-03-19 09:48:38','703505081444930','CLAUDIA BENTO DA SILVA','32 ano(s), 3 meses e 20 dia(s).','C18  - Neoplasia malígna do colon','2026-06-09 07:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623169','2026-03-19 09:52:03','700304985310133','REGINA CELIA GALDINO DA SILVA RECALDES','65 ano(s), 1 meses e 11 dia(s).','Z129 - Exame especial de rastreamento de neoplasia nao especificada','2026-06-09 08:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623264','2026-03-19 10:01:21','708008345335721','ROSA LUCIA DA SILVA','66 ano(s), 6 meses e 25 dia(s).','K529 - Gastroenterite e colite nao-infecciosas, nao especificadas','2026-06-09 08:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623293','2026-03-19 10:04:19','704201718538389','MARIA DE FATIMA LOPES DE OLIVEIRA','58 ano(s), 3 meses e 6 dia(s).','K634 - Enteroptose','2026-06-09 08:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623435','2026-03-19 10:20:40','706403154942882','JENIFFER BATISTA DOS SANTOS','41 ano(s), 9 meses e 9 dia(s).','K922 - Hemorragia gastrointestinal, sem outra especificaçao','2026-06-09 08:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623509','2026-03-19 10:28:11','898000458186734','IARA DE ANDRADE REZENDE COSTA','71 ano(s), 4 meses e 11 dia(s).','Z12  - Exame especial de rastreamento (screening) de neoplasias','2026-06-09 08:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623601','2026-03-19 10:35:58','700000821003806','JOANA D ARC DE LIMA GUSTAVO','62 ano(s), 11 meses e 6 dia(s).','R10  - Dor abdominal e pélvica','2026-06-09 08:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623677','2026-03-19 10:44:20','700408923579646','DILZA MARIA RODRIGUES','73 ano(s), 4 meses e 8 dia(s).','K592 - Cólon neurogenico nao classificado em outra parte','2026-06-09 09:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623718','2026-03-19 10:48:31','706703564481319','NATASHA VENANCIO FERREIRA DE PAULA','32 ano(s), 4 meses e 26 dia(s).','K529 - Gastroenterite e colite nao-infecciosas, nao especificadas','2026-06-09 09:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7623886','2026-03-19 11:03:23','708607570678081','IDE PENA DA SILVEIRA','63 ano(s), 1 meses e 0 dia(s).','K922 - Hemorragia gastrointestinal, sem outra especificaçao','2026-06-09 09:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7624282','2026-03-19 11:51:46','706007302459545','SULAMITA BORGES DE SOUZA','38 ano(s), 2 meses e 28 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 09:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7624979','2026-03-19 13:49:20','898000206010232','JANE BARROS DE SOUZA','68 ano(s), 0 meses e 29 dia(s).','C50  - Neoplasia malígna da mama','2026-06-09 09:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7625847','2026-03-19 15:23:03','708701165877094','MARLY DOS SANTOS','64 ano(s), 5 meses e 10 dia(s).','K57  - Doença diverticular do intestino','2026-06-09 09:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7626227','2026-03-19 15:57:26','705409460858998','SONIA MARIA DA SILVA BAHIA','62 ano(s), 2 meses e 25 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 10:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7626278','2026-03-19 16:03:03','706805253897328','MARIA DE LOURDES PINTO MAIA','67 ano(s), 4 meses e 6 dia(s).','Z80  - História familiar de neoplasia malígna','2026-06-09 10:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7626535','2026-03-19 16:37:54','700807485554481','ANDREIA DE FREITAS PIRES','48 ano(s), 7 meses e 25 dia(s).','K602 - Fissura anal, nao especificada','2026-06-09 10:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628444','2026-03-20 10:45:06','704702763818332','NEIDE APARECIDA DE ALMEIDA','63 ano(s), 6 meses e 23 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 10:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628575','2026-03-20 10:57:26','704201207472889','ELZIMAR DE FATIMA ALBERNAZ JUNQUEIRA','61 ano(s), 0 meses e 4 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 11:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628605','2026-03-20 10:59:59','700008896091009','LILIA DA SILVA CARDOSO SA','57 ano(s), 11 meses e 14 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 11:10:00','Agendada');

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('EXAME','Colonoscopia com Biópsia','7628653','2026-03-20 11:04:54','703209656514499','PEDRO SOARES','74 ano(s), 6 meses e 15 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 11:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628674','2026-03-20 11:06:42','708802724323710','MIGUEL ALVES VIANA','60 ano(s), 11 meses e 27 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 11:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628688','2026-03-20 11:08:20','700006695920707','FABIOLA DOS SANTOS SILVA FLAUSINO','45 ano(s), 2 meses e 20 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 11:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628719','2026-03-20 11:10:31','700005466208609','RITA GOMES PINTO MONTALVAO','64 ano(s), 0 meses e 19 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-09 11:50:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez em Ortopedia - Joelho (Adulto)','7813117','2026-05-13 11:14:07','700004438553401','CARLOS ALBERTO CESAR GONDIM','64 ano(s), 4 meses e 24 dia(s).','M170 - Gonartrose primária bilateral','2026-06-09 13:04:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez - Coloproctologia (Oncologia)','7830794','2026-05-18 11:58:50','705602435017014','ANGELA AUXILIADORA DE JESUS','54 ano(s), 0 meses e 28 dia(s).','C18  - Neoplasia malígna do colon','2026-06-09 13:30:00','Agendada'),
  ('EXAME','Angiorressonância Cerebral','7727657','2026-04-16 16:01:28','700604919817370','MARCELO RONDINELI COSTA BERNARDO','47 ano(s), 5 meses e 18 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-09 07:40:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7585879','2026-03-10 09:43:55','700702900758880','CLAUDIA MARIA FERREIRA DA SILVA','48 ano(s), 6 meses e 0 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-09 10:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7586100','2026-03-10 10:11:41','706003371156544','MARIA RITA GONCALVES NASCIMENTO','67 ano(s), 2 meses e 4 dia(s).','R51  - Cefaléia','2026-06-09 10:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7586241','2026-03-10 10:23:59','700005509861503','MARIZA SILVA DO NASCIMENTO','69 ano(s), 3 meses e 6 dia(s).','R42  - Tontura e instabilidade','2026-06-09 11:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7586615','2026-03-10 11:12:02','708403733679667','CLAUDIO DE ALMEIDA MACHADO','60 ano(s), 2 meses e 2 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-09 11:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7586805','2026-03-10 11:28:17','706101589117460','ELANE DA ROCHA NASCIMENTO','68 ano(s), 6 meses e 16 dia(s).','H907 - Perda de audiçao unilateral mista, de conduçao e neuro-sensorial, sem restr. audiçao contralateral','2026-06-09 13:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7586536','2026-03-10 11:05:21','700201404106025','ADILSON DA SILVA','70 ano(s), 1 meses e 29 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-09 14:40:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7586919','2026-03-10 11:39:36','708006818919421','JAQUELINE DE CASTRO REZENDE','50 ano(s), 10 meses e 6 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-09 15:15:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7586879','2026-03-10 11:35:31','704600173675125','ZIMAR TEREZA MAIA','50 ano(s), 4 meses e 9 dia(s).','O92  - Outras afecçoes da mama e da lactaçao associadas ao parto','2026-06-09 15:50:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7587784','2026-03-10 13:28:36','700004860991103','VIVIANI ROSA BARBOSA DE SOUZA','49 ano(s), 8 meses e 4 dia(s).','G40  - Epilepsia','2026-06-09 17:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM PNEUMOLOGIA - GERAL','6804102','2025-07-21 13:29:24','700501538196956','MAURO SERGIO MAXIMO','52 ano(s), 8 meses e 13 dia(s).','C34  - Neoplasia malígna dos bronquios e dos pulmoes','2026-06-10 08:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OFTALMOLOGIA - GERAL','7727982','2026-04-16 16:42:30','706403165084181','ANTONIETA MARIA AMANCIO FONTES','77 ano(s), 8 meses e 23 dia(s).','H57  - Transtornos do olho e anexos','2026-06-10 12:50:00','Agendada'),
  ('EXAME','Tomografia Computadorizada de Abdome Total','7865415','2026-05-26 14:00:37','703001895338378','MANOEL TEIXEIRA PEREIRA','67 ano(s), 9 meses e 29 dia(s).','R93  - Achados anormais de exame para diagnóstico por imagem de outras estruturas do corpo','2026-06-10 11:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM ENDOCRINOLOGIA - TIREOIDE','7854407','2026-05-22 16:04:36','700807972178582','EDWIRGE DE OLIVEIRA ALVES','71 ano(s), 0 meses e 24 dia(s).','E07  - Transtornos da tireóide','2026-06-10 10:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA - PEDIATRIA','7764871','2026-04-29 12:42:57','700802414935380','LORENA DA PAIXAO CARDOSO','8 ano(s), 7 meses e 4 dia(s).','R074 - Dor torácica, nao especificada','2026-06-10 14:40:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7666391','2026-03-31 10:03:47','702504347568335','JULIANA BARBOSA DE CARVALHO','46 ano(s), 11 meses e 19 dia(s).','R60  - Edema nao classificado em outra parte','2026-06-10 10:04:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7666464','2026-03-31 10:11:03','700002321237206','LARISSA DAVID BRAGA','35 ano(s), 6 meses e 19 dia(s).','I83  - Varizes dos membros inferiores','2026-06-10 10:08:00','Agendada'),
  ('CONSULTA','CONSULTA EM NEUROLOGIA','7799430','2026-05-09 11:47:33','700407499494242','SONIA MARIA LIBANIO BARBOSA','62 ano(s), 7 meses e 12 dia(s).','G44  - Outras síndromes de algias cefálicas','2026-06-10 08:00:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7313941','2025-12-10 14:07:22','706002895211543','HELENIZETE DA CRUZ SILVA','75 ano(s), 11 meses e 18 dia(s).','I738 - Outras doenças vasculares periféricas especificadas','2026-06-10 12:05:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7313975','2025-12-10 14:11:37','704609653460627','ANTONIO CARLOS COUTINHO DA SILVA','67 ano(s), 7 meses e 15 dia(s).','R609 - Edema nao especificado','2026-06-10 12:10:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7314001','2025-12-10 14:16:00','702603241174242','JOAO BATISTA ESTEVES','75 ano(s), 11 meses e 7 dia(s).','R609 - Edema nao especificado','2026-06-10 12:15:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7314082','2025-12-10 14:26:12','700804498288987','SILVANA CHAGAS PINTO','51 ano(s), 3 meses e 29 dia(s).','M712 - Cisto sinovial do espaço poplíteo [Baker]','2026-06-10 12:20:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7314130','2025-12-10 14:31:45','706803281602726','MARIA APARECIDA GUEDES PEREIRA','83 ano(s), 11 meses e 27 dia(s).','I872 - Insuficiencia venosa (crônica) (periférica)','2026-06-10 12:25:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7314288','2025-12-10 14:48:52','700407444264741','CRISTIANE DOS SANTOS BRANDAO','53 ano(s), 8 meses e 9 dia(s).','I872 - Insuficiencia venosa (crônica) (periférica)','2026-06-10 12:30:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7314405','2025-12-10 15:02:05','705002807762358','ADRIANO RODRIGUES NETO','43 ano(s), 6 meses e 11 dia(s).','I872 - Insuficiencia venosa (crônica) (periférica)','2026-06-10 12:35:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7314613','2025-12-10 15:27:40','706200546783068','DILMA SERRAZINE CARREIRA','58 ano(s), 5 meses e 6 dia(s).','I872 - Insuficiencia venosa (crônica) (periférica)','2026-06-10 12:40:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7330233','2025-12-15 16:11:27','705203463909778','MARIA PRESCILA VENTURA','62 ano(s), 8 meses e 5 dia(s).','I739 - Doenças vasculares periféricas nao especificada','2026-06-10 12:45:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7314317','2025-12-10 14:52:16','708601515665288','EMILIA DE JESUS BALBINO','73 ano(s), 7 meses e 10 dia(s).','I872 - Insuficiencia venosa (crônica) (periférica)','2026-06-10 12:50:00','Agendada'),
  ('EXAME','DOPPLER ARTERIAL DE MMII','7314585','2025-12-10 15:24:28','701406616439835','LUIZ ANTONIO MELLO DOS SANTOS','60 ano(s), 11 meses e 26 dia(s).','I872 - Insuficiencia venosa (crônica) (periférica)','2026-06-10 12:55:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª Vez - Patologia Cirúrgica da Coluna Vertebral (Infantil)','7836680','2026-05-19 12:14:50','706002804918444','MARIA CLARA MARQUES MARINS','13 ano(s), 0 meses e 6 dia(s).','M410 - Escoliose idiopática infantil','2026-06-10 13:00:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - TIREOIDE','7851953','2026-05-22 11:06:10','701208032563912','FERNANDA DE MOURA CHAVES VALADAO','46 ano(s), 3 meses e 4 dia(s).','E07  - Transtornos da tireóide','2026-06-10 10:55:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - TIREOIDE','7852028','2026-05-22 11:13:20','898003935516411','JOANA DARC DA SILVA ESTEVAM','54 ano(s), 7 meses e 2 dia(s).','E07  - Transtornos da tireóide','2026-06-10 10:56:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - TIREOIDE','7852095','2026-05-22 11:19:32','708207171902542','ELISABETE APARECIDA DE MELLO','47 ano(s), 10 meses e 21 dia(s).','E34  - Transtornos endocrinos','2026-06-10 10:58:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - TIREOIDE','7852139','2026-05-22 11:24:30','708002822349924','MARIA DAS GRACAS DE SOUZA NUNES','68 ano(s), 2 meses e 28 dia(s).','E07  - Transtornos da tireóide','2026-06-10 10:59:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - TIREOIDE','7852163','2026-05-22 11:26:49','708102866263110','COSMENCINDA CONCEICAO LOPES MACHADO','71 ano(s), 1 meses e 26 dia(s).','E079 - Transtorno nao especificado da tireóide','2026-06-10 11:00:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - TIREOIDE','7852418','2026-05-22 11:51:21','705202451928275','SILVANA DOS SANTOS SILVA','60 ano(s), 4 meses e 26 dia(s).','E07  - Transtornos da tireóide','2026-06-10 11:01:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - TIREOIDE','7852441','2026-05-22 11:53:04','704109028391750','HOSANA GONCALVES PORTUGAL','42 ano(s), 10 meses e 24 dia(s).','E07  - Transtornos da tireóide','2026-06-10 11:02:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - TIREOIDE','7852517','2026-05-22 12:01:16','708003836505924','MERILEIDY D AVILA DOS SANTOS','65 ano(s), 2 meses e 18 dia(s).','E07  - Transtornos da tireóide','2026-06-10 11:03:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7585365','2026-03-10 08:34:28','704602166631721','IVANILDA ESTEVES DA SILVA','59 ano(s), 9 meses e 22 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-10 08:06:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7567202','2026-03-04 16:08:24','708608007954885','CLAUDIONIRIA DA SILVA CORREA','55 ano(s), 2 meses e 21 dia(s).','R10  - Dor abdominal e pélvica','2026-06-10 07:00:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7567268','2026-03-04 16:16:35','700008883859108','MARIA EMILIA DE BARROS LOIO','62 ano(s), 10 meses e 15 dia(s).','K76  - Doenças do fígado','2026-06-10 07:04:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7567286','2026-03-04 16:19:10','706504330500298','MARIA DA CONCEICAO DE MOURA DOS SANTOS','55 ano(s), 8 meses e 28 dia(s).','R14  - Flatulencia e afecçoes correlatas','2026-06-10 07:08:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7567351','2026-03-04 16:26:40','700202478784427','NILCEA DE OLIVEIRA','72 ano(s), 4 meses e 20 dia(s).','N20  - Calculose do rim e do ureter','2026-06-10 07:12:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7567405','2026-03-04 16:32:20','898003410623355','EDNA REGINA DO PATROCINIO GONTIJO','61 ano(s), 9 meses e 20 dia(s).','R10  - Dor abdominal e pélvica','2026-06-10 07:16:00','Agendada');

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7806570','2026-05-12 09:25:24','706208054059665','ANDREZZA DE OLIVEIRA ESTEVAO','33 ano(s), 10 meses e 6 dia(s).','S93  - Luxaçao entorse e distensao das articulaçoes e dos ligamentos ao nível do tornozelo e do pé','2026-06-10 07:20:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7822283','2026-05-15 08:36:45','708400393784270','CATIA RODRIGUES DE ALMEIDA','58 ano(s), 7 meses e 20 dia(s).','M65  - Sinovite e tenossinovite','2026-06-10 07:28:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7829110','2026-05-18 09:13:53','708009399838927','ANA LUCIA FERREIRA GOMES SILVA','59 ano(s), 10 meses e 17 dia(s).','S93  - Luxaçao entorse e distensao das articulaçoes e dos ligamentos ao nível do tornozelo e do pé','2026-06-10 07:32:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7829262','2026-05-18 09:31:07','704209705269489','CELIA DE OLIVEIRA','68 ano(s), 0 meses e 22 dia(s).','M765 - Tendinite patelar','2026-06-10 07:36:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7597554','2026-03-12 12:17:32','700001552428200','IVONE DE PAULA MONTEIRO','72 ano(s), 9 meses e 16 dia(s).','M759 - Lesao nao especificada do ombro','2026-06-10 07:40:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7597629','2026-03-12 12:27:15','706704793457220','CARLOS AUGUSTO RUFINO','65 ano(s), 11 meses e 8 dia(s).','M75  - Lesoes do ombro','2026-06-10 07:44:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7597683','2026-03-12 12:36:24','708004385394822','ALEXANDRE DA GLORIA OLIVEIRA','51 ano(s), 1 meses e 17 dia(s).','G560 - Síndrome do túnel do carpo','2026-06-10 07:48:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7598687','2026-03-12 14:51:52','705606403726110','WILSON HONORATO DO NASCIMENTO','58 ano(s), 2 meses e 26 dia(s).','M796 - Dor em membro','2026-06-10 07:52:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7632528','2026-03-21 13:58:53','708607154007790','EDILAINE FRANCO GONCALVES','50 ano(s), 2 meses e 12 dia(s).','M752 - Tendinite bicepital','2026-06-10 07:56:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7635559','2026-03-23 11:30:09','700002879220208','LUCIARA BRUNORIO DA SILVEIRA DE JESUS','47 ano(s), 4 meses e 21 dia(s).','G560 - Síndrome do túnel do carpo','2026-06-10 08:00:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7640569','2026-03-24 10:54:30','709004873142014','MARIA DO CARMO DE JESUS','79 ano(s), 4 meses e 14 dia(s).','G560 - Síndrome do túnel do carpo','2026-06-10 08:04:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7654852','2026-03-27 10:17:43','700505119810853','ELIANE DE ANDRADE SILVA','49 ano(s), 7 meses e 28 dia(s).','G560 - Síndrome do túnel do carpo','2026-06-10 08:08:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7654900','2026-03-27 10:23:49','702907576878471','VALERIA PEROZINI AMBROZIO','58 ano(s), 6 meses e 29 dia(s).','M75  - Lesoes do ombro','2026-06-10 08:12:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7706168','2026-04-11 11:06:57','700502133399658','SEBASTIAO LUIZ DE SOUZA','85 ano(s), 9 meses e 0 dia(s).','N39  - Transtornos do trato urinário','2026-06-10 08:20:00','Agendada'),
  ('EXAME','Ultrassonografia de Região Cervical','7585378','2026-03-10 08:37:33','702607726693849','VANISA MARIA MENDES COSTA','57 ano(s), 7 meses e 18 dia(s).','E07  - Transtornos da tireóide','2026-06-10 08:36:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7593956','2026-03-11 15:51:11','704104180739174','DAIANE GOMES DORNELLAS','29 ano(s), 9 meses e 9 dia(s).','G35  - Esclerose múltipla','2026-06-10 15:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7594517','2026-03-11 17:29:31','700609932062161','IRENE MACIEL DE GUSMAO','76 ano(s), 8 meses e 14 dia(s).','H90  - Perda de audiçao por transtorno de conduçao e/ou neuro-sensorial','2026-06-10 16:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7594545','2026-03-11 17:38:47','709500678107770','IGNEZ GUIMARAES VIANNA','86 ano(s), 7 meses e 29 dia(s).','F03  - Demencia nao especificada','2026-06-10 16:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7597821','2026-03-12 12:58:09','700409475520849','KARINNE VALENTE PEREIRA','53 ano(s), 2 meses e 8 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-10 17:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7597898','2026-03-12 13:09:28','706200064023860','FRANCISCO ARLINDO FERNANDES MANSO','68 ano(s), 7 meses e 26 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-10 17:30:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7598900','2026-03-12 15:11:34','708409788066867','MAYARA DA SILVA LORAT','32 ano(s), 7 meses e 21 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-10 19:00:00','Agendada'),
  ('EXAME','DENSITOMETRIA OSSEA - RADIODIAGNOSTICO','7858272','2026-05-25 10:23:35','704107272181180','ELY ARAGAO DA SILVA','90 ano(s), 6 meses e 27 dia(s).','M859 - Transtorno nao especificado da densidade e da estrutura ósseas','2026-06-11 08:59:00','Agendada'),
  ('EXAME','Angiotomografia Coronariana (ambulatorial)','7522595','2026-02-19 15:19:15','702604242468348','AUCINEIA DE SOUZA RIBEIRO','65 ano(s), 3 meses e 8 dia(s).','I25  - Doença isquemica crônica do coraçao','2026-06-11 10:45:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7666679','2026-03-31 10:27:49','704801527634442','RAFAELA APARECIDA MENDES GONCALO DA SILVA','28 ano(s), 0 meses e 3 dia(s).','M796 - Dor em membro','2026-06-11 09:00:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7666751','2026-03-31 10:34:35','705703453795430','MARIA CLAUDETE TEIXEIRA DA SILVA','68 ano(s), 2 meses e 21 dia(s).','I83  - Varizes dos membros inferiores','2026-06-11 09:15:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7666786','2026-03-31 10:37:15','705404466534596','ARMINDA LEANDRA DA SILVA','59 ano(s), 9 meses e 4 dia(s).','M796 - Dor em membro','2026-06-11 09:30:00','Agendada'),
  ('EXAME','DOPPLER VENOSO / ECODOPPLER DE VASOS','7666919','2026-03-31 10:49:22','703604094863536','SANDRA CRISTINA LIMA ERMIDA','57 ano(s), 3 meses e 8 dia(s).','M796 - Dor em membro','2026-06-11 09:45:00','Agendada'),
  ('CONSULTA','CONSULTA EM NEUROLOGIA','7799435','2026-05-09 11:52:54','706002393013346','IVANI DE BARROS','66 ano(s), 4 meses e 1 dia(s).','G44  - Outras síndromes de algias cefálicas','2026-06-11 08:00:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - ABDOME TOTAL','7416595','2026-01-17 12:29:53','706807293881629','DEIVISON MATEUS PAULA E SILVA','17 ano(s), 9 meses e 17 dia(s).','N64  - Doenças da mama','2026-06-11 15:39:00','Agendada'),
  ('EXAME','ULTRASSONOGRAFIA - TIREOIDE','7853202','2026-05-22 13:50:13','700002114124201','SILVANIA AZEVEDO BARBOSA CORREA','52 ano(s), 5 meses e 11 dia(s).','E07  - Transtornos da tireóide','2026-06-11 16:29:00','Agendada'),
  ('CONSULTA','Projeto Acolhe (prevenção da gravidez indesejada)','7843238','2026-05-20 15:12:43','703001864484674','ANDREZA CRISTINA BRAZIEL RODRIGUES','30 ano(s), 7 meses e 9 dia(s).','Z975 - Presença de dispositivo anticoncepcional intra-uterino [diu]','2026-06-11 10:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628743','2026-03-20 11:12:48','708007807902825','WASHINGTON NOGUEIRA NETO','64 ano(s), 6 meses e 10 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 08:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628786','2026-03-20 11:16:30','708001899665722','ROBERTO DOS SANTOS SILVA','75 ano(s), 5 meses e 29 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 08:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628830','2026-03-20 11:20:32','702501370929834','CARLA LEMOS NOBREGA','61 ano(s), 0 meses e 2 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 08:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7628996','2026-03-20 11:36:35','708508324087074','AMAURI CESAR FRANCISCO','57 ano(s), 2 meses e 25 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 09:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7629010','2026-03-20 11:38:32','706208587846061','ELIENE DO NASCIMENTO SANTIAGO MOREIRA','52 ano(s), 7 meses e 22 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 09:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7629085','2026-03-20 11:48:17','704206733996087','MARIA DE LOURDES DANTAS SPIS','63 ano(s), 4 meses e 29 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 09:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7629112','2026-03-20 11:50:50','706409145583188','SOLANGE MARIANO DA MOTA PAES','66 ano(s), 0 meses e 10 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 09:30:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7629155','2026-03-20 11:53:36','709009871258419','PAULO ROBERTO DOS SANTOS','70 ano(s), 3 meses e 18 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 09:40:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7629214','2026-03-20 11:59:32','703403964819900','EYDINAR FIGUEIREDO LEMOS','76 ano(s), 5 meses e 23 dia(s).','Z121 - Exame especial de rastreamento de neoplasia do trato intestinal','2026-06-11 09:50:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7629241','2026-03-20 12:02:10','700607451635766','BELMIRA MARIA NOBREGA DE SOUZA','75 ano(s), 0 meses e 2 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 10:00:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7629254','2026-03-20 12:04:04','702905557827075','MARLI MARIANNO','84 ano(s), 5 meses e 28 dia(s).','K59  - Transtornos funcionais do intestino','2026-06-11 10:10:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7630408','2026-03-20 14:50:35','700609948277960','JOSE FELICIANO DE CARVALHO','71 ano(s), 5 meses e 22 dia(s).','Z121 - Exame especial de rastreamento de neoplasia do trato intestinal','2026-06-11 10:20:00','Agendada'),
  ('EXAME','Colonoscopia com Biópsia','7630452','2026-03-20 14:54:52','708202111916941','ROSENILDA DE AMORIM NEVES','77 ano(s), 1 meses e 7 dia(s).','Z121 - Exame especial de rastreamento de neoplasia do trato intestinal','2026-06-11 10:30:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez em Ortopedia - Joelho (Adulto)','6707912','2025-06-23 10:29:48','705402484041391','ANDERSON OLIVEIRA FERREIRA','45 ano(s), 0 meses e 25 dia(s).','M232 - Transtorno do menisco devido r ruptura ou lesao antiga','2026-06-11 09:50:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584400','2026-03-09 16:56:11','701207085850214','TELCO GERALDO DE OLIVEIRA','51 ano(s), 3 meses e 18 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:04:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584408','2026-03-09 16:57:35','898003925569741','CINTIA TAVARES CHAUL CAMILLO','50 ano(s), 8 meses e 0 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:05:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584411','2026-03-09 16:58:11','705003846439652','CARLOS EDUARDO VIANA DOS SANTOS','48 ano(s), 0 meses e 21 dia(s).','I50  - Insuficiencia cardíaca','2026-06-11 08:06:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584417','2026-03-09 16:59:22','702503385223934','LAURO FERREIRA DE SOUZA','65 ano(s), 1 meses e 26 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:07:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584426','2026-03-09 17:01:23','704308558344099','VALCIRLENE DE SOUZA ROMANO','55 ano(s), 7 meses e 27 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:08:00','Agendada');

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584431','2026-03-09 17:02:22','898003434142535','FLAVIA RODRIGUES','38 ano(s), 2 meses e 14 dia(s).','E11  - Diabetes mellitus nao-insulino-dependente','2026-06-11 08:09:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584443','2026-03-09 17:04:23','704201293344589','SILVANA DE PAULA DA SILVA MARTINS','46 ano(s), 9 meses e 23 dia(s).','I50  - Insuficiencia cardíaca','2026-06-11 08:10:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584444','2026-03-09 17:04:27','898003420323141','ALEXSANDRA MAZZACOLI DE MOURA','69 ano(s), 0 meses e 25 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:11:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584455','2026-03-09 17:07:47','706508309348699','GILMAR CEZAR DE CARVALHO','60 ano(s), 4 meses e 20 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:12:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584463','2026-03-09 17:09:35','703200605168196','ADRIANA CRISTINA TEIXEIRA NUNES','50 ano(s), 10 meses e 27 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:13:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584475','2026-03-09 17:11:51','700007592065503','MARIA CRISTINA PEREIRA DOS SANTOS','62 ano(s), 0 meses e 4 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:14:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7584487','2026-03-09 17:15:54','708102114220140','MIRIAM DA SILVA MONTEIRO','62 ano(s), 9 meses e 26 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:15:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7586913','2026-03-10 11:38:59','700001797772702','RICARDO DA SILVA SOUZA','52 ano(s), 5 meses e 7 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:16:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7586932','2026-03-10 11:40:41','700902914014399','CLEIDE MARIA DE SOUZA','52 ano(s), 5 meses e 3 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:17:00','Agendada'),
  ('EXAME','ECOCARDIOGRAFIA TRANSTORACICA','7586982','2026-03-10 11:44:51','707005877168934','FABIO DA SILVA SANTOS','51 ano(s), 0 meses e 4 dia(s).','I10  - Hipertensao essencial (primária)','2026-06-11 08:18:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7571117','2026-03-05 13:59:10','704004376297264','FABIANO MAGALHAES DE MORAES','54 ano(s), 10 meses e 28 dia(s).','K70  - Doença álcoolica do fígado','2026-06-11 07:00:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7571145','2026-03-05 14:03:10','704002815272269','NELCI SILVA DOS SANTOS','61 ano(s), 3 meses e 27 dia(s).','R10  - Dor abdominal e pélvica','2026-06-11 07:04:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7571182','2026-03-05 14:07:10','704208767509687','NATALIA GUIMARAES MACHADO','37 ano(s), 2 meses e 22 dia(s).','R10  - Dor abdominal e pélvica','2026-06-11 07:08:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7571202','2026-03-05 14:09:20','700502417821060','TANIA REGINA AMANCIO DA COSTA','58 ano(s), 4 meses e 9 dia(s).','R10  - Dor abdominal e pélvica','2026-06-11 07:12:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7571297','2026-03-05 14:20:58','702105706306591','JUMARILZA TOLEDO PIZA','59 ano(s), 2 meses e 9 dia(s).','K80  - Colelitiase','2026-06-11 07:16:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7571325','2026-03-05 14:23:46','700904981825192','HELENA DE FATIMA SILVA MACHADO DE SOUZA','69 ano(s), 1 meses e 26 dia(s).','K43  - Hérnia ventral','2026-06-11 07:20:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7571807','2026-03-05 15:17:02','706405680916681','RENATO MARIANO DE OLIVEIRA','56 ano(s), 3 meses e 16 dia(s).','R10  - Dor abdominal e pélvica','2026-06-11 07:24:00','Agendada'),
  ('EXAME','Ultrassonografia de Abdômen Total','7571895','2026-03-05 15:23:34','703209681370594','REJANE MEDEIROS DOS SANTOS VIEIRA','49 ano(s), 9 meses e 16 dia(s).','R14  - Flatulencia e afecçoes correlatas','2026-06-11 07:28:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7706171','2026-04-11 11:09:09','700001307545208','VICTOR HUGO BRANDAO EMILIO','16 ano(s), 11 meses e 20 dia(s).','N39  - Transtornos do trato urinário','2026-06-11 07:32:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7706174','2026-04-11 11:11:19','703004867038872','CLAUDINEI MOREIRA','56 ano(s), 11 meses e 7 dia(s).','N39  - Transtornos do trato urinário','2026-06-11 07:36:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7706176','2026-04-11 11:14:14','709205289993939','WESLEI CABRAL DOS PASSOS','35 ano(s), 1 meses e 29 dia(s).','N39  - Transtornos do trato urinário','2026-06-11 07:40:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7706185','2026-04-11 11:22:42','700008236088809','CILAS DOMS SERRAZINE','76 ano(s), 0 meses e 19 dia(s).','N39  - Transtornos do trato urinário','2026-06-11 07:44:00','Agendada'),
  ('EXAME','Ultrassonografia de Aparelho Urinário','7706194','2026-04-11 11:28:02','702409059930824','LARAH DE OLIVEIRA MOREIRA VIEIRA','18 ano(s), 4 meses e 8 dia(s).','N398 - Outros transtornos especificados do aparelho urinário','2026-06-11 07:48:00','Agendada'),
  ('EXAME','Ultrassonografia de Tireóide','7513787','2026-02-13 10:55:33','700007972559710','ELIZABETH SHEILA DUARTE DE ALMEIDA','70 ano(s), 2 meses e 5 dia(s).','E07  - Transtornos da tireóide','2026-06-11 07:52:00','Agendada'),
  ('EXAME','Ultrassonografia de Tireóide','7513813','2026-02-13 10:59:47','700001316202105','INERIA JOSE DA SILVA PINTO','57 ano(s), 1 meses e 22 dia(s).','E07  - Transtornos da tireóide','2026-06-11 07:56:00','Agendada'),
  ('EXAME','Ultrassonografia de Tireóide','7513872','2026-02-13 11:09:38','707601252197698','SIMONE APARECIDA DO VALE DE SOUZA','61 ano(s), 1 meses e 2 dia(s).','E07  - Transtornos da tireóide','2026-06-11 08:00:00','Agendada'),
  ('EXAME','Ultrassonografia de Tireóide','7513886','2026-02-13 11:12:22','702304192387512','LUCIMAR MACHADO','60 ano(s), 7 meses e 19 dia(s).','E07  - Transtornos da tireóide','2026-06-11 08:04:00','Agendada'),
  ('EXAME','Ultrassonografia de Tireóide','7513901','2026-02-13 11:14:16','700208956441726','MARINA DOS SANTOS ADELINO VITORINO','40 ano(s), 4 meses e 0 dia(s).','E07  - Transtornos da tireóide','2026-06-11 08:08:00','Agendada'),
  ('EXAME','Ultrassonografia de Tireóide','7513942','2026-02-13 11:20:26','706203074816165','PATRICIA ROSALINA GOMES','49 ano(s), 7 meses e 12 dia(s).','E07  - Transtornos da tireóide','2026-06-11 08:12:00','Agendada'),
  ('EXAME','Ultrassonogradia de Partes Moles','7711490','2026-04-13 15:30:21','706400635784289','MARILIA DA SILVA LOTITO','50 ano(s), 2 meses e 7 dia(s).','M796 - Dor em membro','2026-06-11 08:44:00','Agendada'),
  ('EXAME','Ultrassonogradia de Partes Moles','7787602','2026-05-06 14:43:21','704209212482282','NATALIA DA SILVA RODRIGUES','28 ano(s), 5 meses e 5 dia(s).','L989 - Afecçoes da pele e do tecido subcutâneo, nao especificados','2026-06-11 08:48:00','Agendada'),
  ('EXAME','Ultrassonografia de Parede Abdominal','7597230','2026-03-12 11:42:20','705803414247737','ROSILENE OLIVEIRA DA SILVA DE SOUZA','56 ano(s), 11 meses e 20 dia(s).','T81  - Complicaçoes de procedimentos nao classificadas em outra parte','2026-06-11 08:56:00','Agendada'),
  ('EXAME','Ultrassonografia de Parede Abdominal','7630301','2026-03-20 14:40:50','702304078782920','LUIS DE OLIVEIRA','67 ano(s), 0 meses e 27 dia(s).','K40  - Hérnia inguinal','2026-06-11 09:00:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7655125','2026-03-27 10:49:40','704200273019487','MARCIA CRISTINA FRANCISCO VELOZO','60 ano(s), 4 meses e 15 dia(s).','M65  - Sinovite e tenossinovite','2026-06-11 09:04:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7655884','2026-03-27 12:20:45','709604640469171','FABIOLA DOS SANTOS','46 ano(s), 5 meses e 18 dia(s).','M75  - Lesoes do ombro','2026-06-11 09:08:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7655913','2026-03-27 12:28:52','700008424712301','ANTONIO BEVILAQUA','55 ano(s), 11 meses e 20 dia(s).','M75  - Lesoes do ombro','2026-06-11 09:12:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7656504','2026-03-27 14:12:39','707005829691835','CATIA RODRIGUES DA SILVA','50 ano(s), 4 meses e 10 dia(s).','M75  - Lesoes do ombro','2026-06-11 09:16:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7679803','2026-04-04 13:21:01','706407683622982','JOANA NOGUEIRA ALTINO DE BARROS','54 ano(s), 11 meses e 6 dia(s).','M75  - Lesoes do ombro','2026-06-11 09:20:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7679805','2026-04-04 13:23:36','708206158901944','GABRIEL MARCOS DA SILVA','49 ano(s), 3 meses e 3 dia(s).','M75  - Lesoes do ombro','2026-06-11 09:24:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7679807','2026-04-04 13:25:43','709206264340436','INEZ MORAES DE ANDRADE','71 ano(s), 4 meses e 9 dia(s).','M75  - Lesoes do ombro','2026-06-11 09:28:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7679812','2026-04-04 13:28:15','704605159051327','RITA DA CRUZ MORGADO','85 ano(s), 7 meses e 8 dia(s).','M75  - Lesoes do ombro','2026-06-11 09:32:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7679821','2026-04-04 13:33:51','705003221055450','MARIA APARECIDA INGUINA DA SILVA','66 ano(s), 7 meses e 19 dia(s).','M75  - Lesoes do ombro','2026-06-11 09:36:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Superiores','7679826','2026-04-04 13:39:28','700502991977957','CARLA VIEIRA SOARES','58 ano(s), 10 meses e 27 dia(s).','M75  - Lesoes do ombro','2026-06-11 09:40:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7834741','2026-05-19 09:26:26','704609181990423','JUSSARA LUCIANA DIAS DE PAULA','59 ano(s), 1 meses e 12 dia(s).','M659 - Sinovite e tenossinovite nao especificadas','2026-06-11 09:44:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7834975','2026-05-19 09:47:23','704009325669967','CEILA GOMES','59 ano(s), 7 meses e 13 dia(s).','M767 - Tendinite do perôneo','2026-06-11 09:48:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7835311','2026-05-19 10:16:08','702401077636528','CARMEM DE MELO INACIO ANTONIO','60 ano(s), 8 meses e 15 dia(s).','M722 - Fibromatose da fáscia plantar','2026-06-11 09:52:00','Agendada'),
  ('EXAME','Ultrassonografia de Articulações de Membros Inferiores','7836593','2026-05-19 12:04:46','702802170896361','MARIA LUCIA DE PAULA GURGEL DE SOUZA','60 ano(s), 9 meses e 5 dia(s).','M765 - Tendinite patelar','2026-06-11 09:56:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7605581','2026-03-14 08:45:17','700006095167706','SANDRA ROSANGELA LAUREANO GUEDES','64 ano(s), 5 meses e 26 dia(s).','R26  - Anormalidades da marcha e da mobilidade','2026-06-11 14:05:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7605592','2026-03-14 08:59:15','709205224717538','ADEMILTON ROSA DA SILVA','45 ano(s), 11 meses e 11 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-11 14:40:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7605614','2026-03-14 09:18:24','702805623119860','GERSON DA SILVA','69 ano(s), 3 meses e 28 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-11 15:15:00','Agendada');

INSERT INTO public.agendamentos (tipo, recurso, id_solicitacao, data_solicitacao, cns, paciente, idade, cid, agendado_para, situacao) VALUES
  ('EXAME','Ressonância Magnética de Crânio','7605626','2026-03-14 09:23:28','706704545150516','MARIA ANTONIA SILVA','71 ano(s), 5 meses e 28 dia(s).','I64  - Acidente vascular cerebral, nao especificado como hemorrágico ou isquemico','2026-06-11 15:50:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7605686','2026-03-14 10:00:44','703406221350617','NEOCRECILDO GOMES DE ALMEIDA','59 ano(s), 4 meses e 8 dia(s).','R930 - Achados anormais de exames para diagnóstico por imagem do crânio e da cabeça n class. em outra parte','2026-06-11 16:25:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7605732','2026-03-14 10:37:53','700509504964958','MARIA APRIGIO DE ALMEIDA','72 ano(s), 5 meses e 2 dia(s).','R413 - Outra amnésia','2026-06-11 17:00:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7605772','2026-03-14 11:01:27','706802288137823','PAULA GABRIELA DA SILVA','45 ano(s), 1 meses e 2 dia(s).','R41  - Outros sintomas e sinais relativos a funçao cognitiva e a consciencia','2026-06-11 17:35:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7605785','2026-03-14 11:05:51','702109759923793','SANDOVAL CORREA DUARTE','71 ano(s), 4 meses e 29 dia(s).','G20  - Doença de Parkinson','2026-06-11 18:10:00','Agendada'),
  ('EXAME','Ressonância Magnética de Crânio','7605799','2026-03-14 11:10:03','898003464703128','ALEXSANDRE DOS SANTOS','52 ano(s), 9 meses e 22 dia(s).','R51  - Cefaléia','2026-06-11 18:45:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA','7676334','2026-04-02 09:45:50','898004030132245','RUI CUNHA','83 ano(s), 2 meses e 27 dia(s).','I51  - Doenças cardíacas maldefinidas','2026-06-11 09:20:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA','7675251','2026-04-01 17:21:49','708205112985443','MARIA JOSE DO CARMO SOUZA','80 ano(s), 11 meses e 8 dia(s).','I51  - Doenças cardíacas maldefinidas','2026-06-11 09:30:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA','7675254','2026-04-01 17:23:24','708005839516120','WAGNER GERALDO AMENTA','78 ano(s), 8 meses e 8 dia(s).','I51  - Doenças cardíacas maldefinidas','2026-06-11 09:40:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA','7652853','2026-03-26 16:01:54','707401046163572','MARILZA SANTOS RIBEIRO','78 ano(s), 7 meses e 20 dia(s).','I51  - Doenças cardíacas maldefinidas','2026-06-11 09:45:00','Agendada'),
  ('CONSULTA','CONSULTA EM CARDIOLOGIA','7663467','2026-03-30 14:29:23','709608611155273','LUIZ CARLOS DE MELO','77 ano(s), 1 meses e 14 dia(s).','I51  - Doenças cardíacas maldefinidas','2026-06-11 10:00:00','Agendada'),
  ('CONSULTA','CONSULTA EM OTORRINOLARINGOLOGIA','6931256','2025-08-25 11:52:19','700003885866204','ROGERIO DE OLIVEIRA SOUZA','53 ano(s), 8 meses e 13 dia(s).','H720 - Perfuraçao central da membrana do tímpano','2026-06-11 08:05:00','Agendada'),
  ('CONSULTA','Ambulatório 1ª vez - Hematologia (Oncologia)','7778463','2026-05-04 17:02:08','898003915250378','EDIMAR CANDIDO DE VASCONCELOS','74 ano(s), 5 meses e 17 dia(s).','C82  - Linfoma nao-Hodgkin folicular (nodular)','2026-06-11 13:00:00','Agendada'),
  ('EXAME','Elastografia Hepática Transitória','7092545','2025-10-08 11:09:51','702401523091026','DANIELE CRISTINA DOS SANTOS PINTO RODRIGUES','50 ano(s), 9 meses e 17 dia(s).','K760 - Degeneraçao gordurosa do fígado nao classificada em outra parte','2026-06-11 07:00:00','Agendada');
-- ─────────────────────────────────────────────
-- FIM DO SCRIPT
-- Total: 464 registros importados
-- ─────────────────────────────────────────────
