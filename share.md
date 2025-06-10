string[] lines = await File.ReadAllLinesAsync(sqlPath);

// 1. First line: comma-separated axis and category labels: e.g. "Week,Rec Breaks,Product"
string[] parts = lines[0].Split(',');
var xLabel = parts.ElementAtOrDefault(0)?.Trim();
var yLabel = parts.ElementAtOrDefault(1)?.Trim();
var category = parts.ElementAtOrDefault(2)?.Trim();

// 2. Optional summary in curly braces on second line: "{This is an optional summary…}"
string? summary = null;
if (lines.Length > 1 && lines[1].Trim().StartsWith("{") && lines[1].Trim().EndsWith("}"))
{
    summary = lines[1].Trim().Trim('{', '}').Trim();
}

// 3. The actual SQL starts from either line 1 (no summary) or line 2
int sqlStart = summary is null ? 1 : 2;
string rawSql = string.Join("\n", lines.Skip(sqlStart));
