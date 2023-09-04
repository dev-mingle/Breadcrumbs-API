# Breadcrumbs-API
[팀 과제] 노션에서 브로드 크럼스(Breadcrumbs) 만들기
+ 페이지 정보 조회 API: 특정 페이지의 정보를 조회할 수 있는 API를 구현하세요.
+ [전체 로직URL](https://github.com/GwanUk/WantedAssignment)

<br>

## 테이블 구조
### 테이블명: Page

|속성|타입|조건|
|---|---|---|
|pageId|bigint|PK|
|title|varchar||
|content|varchar||
|parentId|bigint||

<br>

## 페이지 조회 API
#### API
```
/api/v1/post 
```
#### Request
```
{
  "pageId" : 1
}
```
#### Response
```java
{
  "pageId" : 4,
  "title" : "page-d",
  "content" : "content-d"
  "subPages" : ["page-e", "page-f"],
  "breadcrumbs" : ["page-a", "page-b", "page-c", "page-d"]
}
```

<br>

## 비지니스 로직
#### PageService.java
```
public Page findPage(Long pageId){
    PageDao pageDao = new PageDao();
    List<PageDto> dtoList = pageDao.findById(pageId);
    return createPage(dtoList);
}
```
#### PageDao.java
+ SQL 쿼리는 WITH RECURSIVE를 사용하여 페이지의 계층 구조를 표현합니다.
+ 먼저, pageId가 일치하는 최상위 페이지를 선택하고, 그 하위 페이지들을 재귀적으로 가져옵니다. 그리고 최상위 페이지와 그 하위 페이지들을 모두 선택한 후, parent_id가 일치하는 페이지들을 추가로 선택합니다.
+ 이렇게 가져온 페이지 정보들은 while 문을 사용하여 PageDto 객체에 저장된 후, 리스트에 추가됩니다.
+ 그리고 마지막으로, 리스트를 반환합니다.
```
public class PageDao {
    public List<PageDto> findById(Long pageId) {
        String sql = """
                WITH RECURSIVE hierarchy_pages (page_id, title, content, parent_id)
                AS
                (
                    SELECT page_id, title, content, parent_id
                    FROM pages.pages
                    WHERE page_id = ?
                    UNION ALL
                    SELECT c.page_id, c.title, c.content, c.parent_id
                    FROM pages.pages c
                    JOIN hierarchy_pages p
                    ON c.page_id = p.parent_id
                )
                
                SELECT page_id, title, content, parent_id
                FROM hierarchy_pages
                
                UNION ALL
                
                (SELECT page_id, title, content, parent_id
                FROM pages.pages
                WHERE parent_id = ?
                ORDER BY page_id);
                """;

        Connection conn = null;
        PreparedStatement stmt = null;
        ResultSet rs = null;

        try {
            conn = DBConnection.getConnection();
            stmt = conn.prepareStatement(sql);
            stmt.setLong(1, pageId);
            stmt.setLong(2, pageId);
            rs = stmt.executeQuery();

            List<PageDto> result = new ArrayList<>();
            while (rs.next()) {
                PageDto dto = new PageDto();
                dto.setPageId(rs.getLong("page_id"));
                dto.setTitle(rs.getString("title"));
                dto.setContent(rs.getString("content"));
                dto.setParentId(rs.getObject("parent_id", Long.class));
                result.add(dto);
            }

            return result;
        } catch (SQLException e) {
            throw new RuntimeException(e);
        } finally {
            close(conn, stmt, rs);
        }
    }
}
```

<br>
        
## 제출하신 과제에 대해서 설명해주세요.
