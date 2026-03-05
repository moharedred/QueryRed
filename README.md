#"Sheet1 Sheet" = #"Source"{0}[Data],
    #"Promoted Headers" = Table.PromoteHeaders(#"Sheet1 Sheet", [PromoteAllScalars=true]),

    #"Colonnes Presentes" = Table.ColumnNames(#"Promoted Headers"),
    #"Colonnes Simples" = List.Select(
        #"Colonnes Presentes",
        (col) =>
            let v = try Table.Column(#"Promoted Headers", col){0} otherwise null
            in not (Value.Is(v, type table) or Value.Is(v, type list) or Value.Is(v, type record))
    ),
    #"Remove Structured" = Table.SelectColumns(#"Promoted Headers", #"Colonnes Simples"),

    #"Cols Presents 2" = Table.ColumnNames(#"Remove Structured"),
    #"Transform Safe" = List.Select(
        {
            {"Nom de la tâche", type text}, {"Durée", type text}, {"Travail réel", type text},
            {"Type_Evènement", type text}, {"indicateur retard", type text}, {"Variation de fin", type text},
            {"Fin", type text}, {"Fin de référence", type text}, {"Charge totale", type text},
            {"Avancement (KPI 2.2)", type number}, {"Consommé", type text}, {"RAF", type text},
            {"Engagé", type text}, {"Go Vs Activités DED (KPI 2.0)", Int64.Type},
            {"Dérive budgétaire (KPI 2.4)", type text}, {"dérive / budget (KPI 2.4)", type number}
        },
        each List.Contains(#"Cols Presents 2", _{0})
    ),
    #"Changed Type" = Table.TransformColumnTypes(#"Remove Structured", #"Transform Safe"),

    #"Nom Domaine" = #"Changed Type"{0}[Nom de la tâche],
    #"Sans Ligne Domaine" = Table.Skip(#"Changed Type", 1),
    #"Sans Lignes Vides" = Table.SelectRows(#"Sans Ligne Domaine", each
        [Nom de la tâche] <> null and Text.Trim(Text.From([Nom de la tâche])) <> ""
    ),

    #"Add Domaine" = Table.AddColumn(#"Sans Lignes Vides", "Domaine", each #"Nom Domaine"),
    #"Reorder Domaine" = Table.ReorderColumns(#"Add Domaine",
        {"Domaine"} & List.RemoveItems(Table.ColumnNames(#"Add Domaine"), {"Domaine"})
    ),

    #"GetNumber" = (txt as any) as any =>
        if txt = null then null
        else
            let
                s = Text.Remove(Text.Trim(Text.From(txt)), {" "}),
                s2 = Text.Replace(s, ",", "."),
                chars = Text.ToList(s2),
                digits = {"0","1","2","3","4","5","6","7","8","9"},
                startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})),
                charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx),
                endIdx = List.PositionOfAny(charsFromStart, {"j","J","?","h","d"}),
                numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx),
                numStr = Text.Combine(numChars)
            in
                if numStr = "" or numStr = "-" then null
                else try Number.FromText(numStr, "en-US") otherwise null,

    #"Cols Pour Nettoyage" = Table.ColumnNames(#"Reorder Domaine"),
    #"Nettoyer Colonnes" = Table.TransformColumns(#"Reorder Domaine",
        List.Select(
            {
                {"Travail réel",                each #"GetNumber"(_), type number},
                {"Durée",                       each #"GetNumber"(_), type number},
                {"Charge totale",               each #"GetNumber"(_), type number},
                {"Consommé",                    each #"GetNumber"(_), type number},
                {"RAF",                         each #"GetNumber"(_), type number},
                {"Engagé",                      each #"GetNumber"(_), type number},
                {"Dérive budgétaire (KPI 2.4)", each #"GetNumber"(_), type number},
                {"Variation de fin",            each #"GetNumber"(_), type number}
            },
            each List.Contains(#"Cols Pour Nettoyage", _{0})
        )
    ),

    #"Cols Nettoyees" = Table.ColumnNames(#"Nettoyer Colonnes"),
    #"Arrondi Pourcentages" = Table.TransformColumns(#"Nettoyer Colonnes",
        List.Select(
            {
                {"Avancement (KPI 2.2)",        each if _ = null then null else Number.Round(_, 2), type number},
                {"dérive / budget (KPI 2.4)",   each if _ = null then null else Number.Round(_, 2), type number}
            },
            each List.Contains(#"Cols Nettoyees", _{0})
        )
    ),

    #"Nettoyer Type" = if List.Contains(Table.ColumnNames(#"Arrondi Pourcentages"), "Type_Evènement")
        then Table.TransformColumns(#"Arrondi Pourcentages", {{
            "Type_Evènement",
            each if _ = null then null else Text.Lower(Text.Trim(Text.From(_))),
            type text
        }})
        else #"Arrondi Pourcentages",

    #"Add Index" = Table.AddIndexColumn(#"Nettoyer Type", "Index", 0, 1),
    #"Add Niveau" = Table.AddColumn(#"Add Index", "Niveau", each
        let
            raw = [Nom de la tâche],
            nom = if raw = null then "" else if Value.Is(raw, type text) then raw else Text.From(raw),
            espaces = Text.Length(nom) - Text.Length(Text.TrimStart(nom)),
            niveauBrut = Number.IntegerDivide(espaces, 3) + 1
        in
            if niveauBrut < 1 then 1 else if niveauBrut > 12 then 12 else niveauBrut
    ),

    #"rows" = Table.ToRecords(#"Add Niveau"),
    #"nbRows" = List.Count(#"rows"),
    #"Profondeur Min" = List.Min(List.Transform(#"rows", each try Record.Field(_, "Niveau") otherwise null)),

    #"Build Result" = List.Accumulate(
        {0..#"nbRows"-1},
        [compteurs = List.Repeat({0}, 12), resultats = {}],
        (state, i) =>
            let
                row = #"rows"{i},
                niveauRaw = try Record.Field(row, "Niveau") otherwise null,
                niveau = if niveauRaw = null then 1
                         else if niveauRaw < 1 then 1
                         else if niveauRaw > 12 then 12
                         else niveauRaw,
                c = state[compteurs],
                nouveauxCompteurs = List.Generate(
                    () => [n = 0], each [n] < 12, each [n = [n] + 1],
                    each
                        if [n] + 1 = niveau then c{[n]} + 1
                        else if [n] + 1 > niveau then 0
                        else c{[n]}
                ),
                chemin = Text.Combine(
                    List.Transform({0..niveau-1}, each Text.From(nouveauxCompteurs{_})), "."
                ),
                rowClean = Record.SelectFields(row,
                    List.Select(Record.FieldNames(row), each _ <> "Chemin" and _ <> "Profondeur")
                ),
                newRow = Record.AddField(Record.AddField(rowClean, "Chemin", chemin), "Profondeur", niveau)
            in
                [compteurs = nouveauxCompteurs, resultats = state[resultats] & {newRow}]
    ),

    #"Final Table" = Table.FromRecords(#"Build Result"[resultats]),

    #"Add Calculs" = Table.AddColumn(#"Final Table", "tmp", each
        let
            idx = [Index],
            niveau = [Niveau]
        in
        if niveau <> #"Profondeur Min" then
            [
                Evenement_charge = null, Evenement_delai = null, Evenement_perimetre = null,
                Impact_coût = null, Impact_planning = null,
                Impact_coût_charge = null, Impact_coût_périmètre = null,
                Impact_planning_délai = null, Impact_planning_périmètre = null
            ]
        else
            let
                nextParent = List.First(
                    List.Select(#"rows", each _[Index] > idx and _[Niveau] = #"Profondeur Min"), null
                ),
                limitIdx = if nextParent = null then #"nbRows" else nextParent[Index],
                children = List.Select(#"rows", each _[Index] > idx and _[Index] < limitIdx),

                GetType = (r as record) => try Record.Field(r, "Type_Evènement") otherwise null,
                IsPerimetre = (r as record) =>
                    let t = GetType(r)
                    in t = "périmetre" or t = "périmètre",

                nbCharge    = List.Count(List.Select(children, each GetType(_) = "charge")),
                nbDelai     = List.Count(List.Select(children, each GetType(_) = "délai")),
                nbPerimetre = List.Count(List.Select(children, each IsPerimetre(_))),

                // Impact_coût = charge + périmetre (Travail réel)
                impactCoutCharge = List.Sum(List.Transform(
                    List.Select(children, each GetType(_) = "charge"),
                    each try Record.Field(_, "Travail réel") otherwise null
                )),
                impactCoutPerimetre = List.Sum(List.Transform(
                    List.Select(children, each IsPerimetre(_)),
                    each try Record.Field(_, "Travail réel") otherwise null
                )),

                // Impact_planning = délai + périmetre (Durée)
                impactPlanningDelai = List.Sum(List.Transform(
                    List.Select(children, each GetType(_) = "délai"),
                    each try Record.Field(_, "Durée") otherwise null
                )),
                impactPlanningPerimetre = List.Sum(List.Transform(
                    List.Select(children, each IsPerimetre(_)),
                    each try Record.Field(_, "Durée") otherwise null
                )),

                impactCout = (if impactCoutCharge = null then 0 else impactCoutCharge) +
                             (if impactCoutPerimetre = null then 0 else impactCoutPerimetre),
                impactPlanning = (if impactPlanningDelai = null then 0 else impactPlanningDelai) +
                                 (if impactPlanningPerimetre = null then 0 else impactPlanningPerimetre)
            in
                [
                    Evenement_charge = nbCharge, Evenement_delai = nbDelai, Evenement_perimetre = nbPerimetre,
                    Impact_coût = impactCout, Impact_planning = impactPlanning,
                    Impact_coût_charge = impactCoutCharge, Impact_coût_périmètre = impactCoutPerimetre,
                    Impact_planning_délai = impactPlanningDelai, Impact_planning_périmètre = impactPlanningPerimetre
                ]
    ),

    #"Expand Cols" = Table.ExpandRecordColumn(#"Add Calculs", "tmp",
        {
            "Evenement_charge", "Evenement_delai", "Evenement_perimetre",
            "Impact_coût", "Impact_planning",
            "Impact_coût_charge", "Impact_coût_périmètre",
            "Impact_planning_délai", "Impact_planning_périmètre"
        }
    ),

    #"Remove Cols" = Table.RemoveColumns(#"Expand Cols", {"Index", "Niveau"}),

    #"Remplace Null Impacts" = Table.TransformColumns(#"Remove Cols",
        List.Select(
            {
                {"Impact_coût",              each if _ = null then 0 else _, type number},
                {"Impact_planning",          each if _ = null then 0 else _, type number},
                {"Impact_coût_charge",       each if _ = null then 0 else _, type number},
                {"Impact_coût_périmètre",    each if _ = null then 0 else _, type number},
                {"Impact_planning_délai",    each if _ = null then 0 else _, type number},
                {"Impact_planning_périmètre",each if _ = null then 0 else _, type number}
            },
            each List.Contains(Table.ColumnNames(#"Remove Cols"), _{0})
        )
    ),

    #"Profondeur Min Final" = List.Min(Table.Column(#"Remplace Null Impacts", "Profondeur")),
    #"Lignes Filtrees" = Table.SelectRows(#"Remplace Null Impacts", each [Profondeur] = #"Profondeur Min Final"),

    #"Format Colonnes" = Table.TransformColumnTypes(#"Lignes Filtrees",
        List.Select(
            {
                {"Travail réel",                  type number},
                {"Durée",                         type number},
                {"Charge totale",                 type number},
                {"Avancement (KPI 2.2)",          type number},
                {"Consommé",                      type number},
                {"RAF",                           type number},
                {"Engagé",                        type number},
                {"Go Vs Activités DED (KPI 2.0)", type number},
                {"Dérive budgétaire (KPI 2.4)",   type number},
                {"dérive / budget (KPI 2.4)",     type number},
                {"Evenement_charge",              type number},
                {"Evenement_delai",               type number},
                {"Evenement_perimetre",           type number},
                {"Impact_coût",                   type number},
                {"Impact_planning",               type number},
                {"Impact_coût_charge",            type number},
                {"Impact_coût_périmètre",         type number},
                {"Impact_planning_délai",         type number},
                {"Impact_planning_périmètre",     type number}
            },
            each List.Contains(Table.ColumnNames(#"Lignes Filtrees"), _{0})
        )
    )

in
    #"Format Colonnes"
