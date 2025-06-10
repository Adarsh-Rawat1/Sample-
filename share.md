string rawSql = await File.ReadAllTextAsync(sqlPath);

// Remove exactly one ';' at end (if present), along with any trailing whitespace
rawSql = rawSql.TrimEnd();  
if (rawSql.EndsWith(";"))
{
    rawSql = rawSql.Substring(0, rawSql.Length - 1).TrimEnd();
}
