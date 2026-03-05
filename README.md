 #"Nettoyer Colonnes" = Table.TransformColumns(#"Reorder Domaine",
        List.Select(
            {
                {"Travail réel", each 
                    let s = Text.Trim(Text.From(_)), s2 = Text.Replace(s, ",", "."), chars = Text.ToList(s2), digits = {"0","1","2","3","4","5","6","7","8","9"}, startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})), charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx), endIdx = List.PositionOfAny(charsFromStart, {" ","j","J","?","h","d"}), numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx), numStr = Text.Combine(numChars) in if numStr = "" then null else try Number.From(numStr) otherwise null
                , type number},
                {"Durée", each 
                    let s = Text.Trim(Text.From(_)), s2 = Text.Replace(s, ",", "."), chars = Text.ToList(s2), digits = {"0","1","2","3","4","5","6","7","8","9"}, startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})), charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx), endIdx = List.PositionOfAny(charsFromStart, {" ","j","J","?","h","d"}), numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx), numStr = Text.Combine(numChars) in if numStr = "" then null else try Number.From(numStr) otherwise null
                , type number},
                {"Charge totale", each 
                    let s = Text.Trim(Text.From(_)), s2 = Text.Replace(s, ",", "."), chars = Text.ToList(s2), digits = {"0","1","2","3","4","5","6","7","8","9"}, startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})), charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx), endIdx = List.PositionOfAny(charsFromStart, {" ","j","J","?","h","d"}), numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx), numStr = Text.Combine(numChars) in if numStr = "" then null else try Number.From(numStr) otherwise null
                , type number},
                {"Consommé", each 
                    let s = Text.Trim(Text.From(_)), s2 = Text.Replace(s, ",", "."), chars = Text.ToList(s2), digits = {"0","1","2","3","4","5","6","7","8","9"}, startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})), charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx), endIdx = List.PositionOfAny(charsFromStart, {" ","j","J","?","h","d"}), numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx), numStr = Text.Combine(numChars) in if numStr = "" then null else try Number.From(numStr) otherwise null
                , type number},
                {"RAF", each 
                    let s = Text.Trim(Text.From(_)), s2 = Text.Replace(s, ",", "."), chars = Text.ToList(s2), digits = {"0","1","2","3","4","5","6","7","8","9"}, startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})), charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx), endIdx = List.PositionOfAny(charsFromStart, {" ","j","J","?","h","d"}), numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx), numStr = Text.Combine(numChars) in if numStr = "" then null else try Number.From(numStr) otherwise null
                , type number},
                {"Engagé", each 
                    let s = Text.Trim(Text.From(_)), s2 = Text.Replace(s, ",", "."), chars = Text.ToList(s2), digits = {"0","1","2","3","4","5","6","7","8","9"}, startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})), charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx), endIdx = List.PositionOfAny(charsFromStart, {" ","j","J","?","h","d"}), numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx), numStr = Text.Combine(numChars) in if numStr = "" then null else try Number.From(numStr) otherwise null
                , type number},
                {"Dérive budgétaire (KPI 2.4)", each 
                    let s = Text.Trim(Text.From(_)), s2 = Text.Replace(s, ",", "."), chars = Text.ToList(s2), digits = {"0","1","2","3","4","5","6","7","8","9"}, startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})), charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx), endIdx = List.PositionOfAny(charsFromStart, {" ","j","J","?","h","d"}), numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx), numStr = Text.Combine(numChars) in if numStr = "" then null else try Number.From(numStr) otherwise null
                , type number},
                {"Variation de fin", each 
                    let s = Text.Trim(Text.From(_)), s2 = Text.Replace(s, ",", "."), chars = Text.ToList(s2), digits = {"0","1","2","3","4","5","6","7","8","9"}, startIdx = List.PositionOfAny(chars, List.Combine({digits, {"-"}})), charsFromStart = if startIdx = -1 then {} else List.Skip(chars, startIdx), endIdx = List.PositionOfAny(charsFromStart, {" ","j","J","?","h","d"}), numChars = if endIdx = -1 then charsFromStart else List.FirstN(charsFromStart, endIdx), numStr = Text.Combine(numChars) in if numStr = "" then null else try Number.From(numStr) otherwise null
                , type number}
            },
            each List.Contains(Table.ColumnNames(#"Reorder Domaine"), _{0})
        )
    ),
