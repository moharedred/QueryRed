#"Debug" = Table.AddColumn(#"Reorder Domaine", "test", each
        let
            s = Text.Trim(Text.From([Travail réel])),
            s2 = Text.Replace(s, ",", "."),
            // Supprimer tout sauf chiffres, point et signe moins
            digits = {"0","1","2","3","4","5","6","7","8","9"},
            chars = Text.ToList(s2),
            startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})),
            charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx),
            // Garder seulement jusqu au premier espace ou lettre
            endIdx = List.PositionOfAny(charsFromStart, {" ","j","J","?","h","d"}),
            numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx),
            numStr = Text.Combine(numChars)
        in
            numStr
    )

in
    #"Debug"
