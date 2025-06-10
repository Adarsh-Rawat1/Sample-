string content = await File.ReadAllTextAsync(sqlPath);
string[] lines = content.Split(new[] { "\r\n", "\n" }, StringSplitOptions.None);

// Parse optional header/metadata
string[] header = lines[0].Split(',');
string xLabel = header.ElementAtOrDefault(0)?.Trim() ?? "";
string yLabel = header.ElementAtOrDefault(1)?.Trim() ?? "";
string category = header.ElementAtOrDefault(2)?.Trim() ?? "";

// Optional summary and SQL start index
string? summary = null;
int sqlStart = 1;
if (lines.Length > 1)
{
    var l2 = lines[1].Trim();
    if (l2.StartsWith("{") && l2.EndsWith("}"))
    {
        summary = l2.Trim('{', '}').Trim();
        sqlStart = 2;
    }
}

// Reconstruct SQL and strip a single trailing semicolon
string rawSql = string.Join("\n", lines.Skip(sqlStart))
                     .TrimEnd();
if (rawSql.EndsWith(";"))
    rawSql = rawSql.Substring(0, rawSql.Length - 1).TrimEnd();
