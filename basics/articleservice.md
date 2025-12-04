---
description: 게시글의 비즈니스 로직(CRUD)를 처리하는 클래스입니다.
---

# ArticleService



```java
@Slf4j
@Service
@RequiredArgsConstructor
public class ArticleService {
    private final ArticleMapper articleMapper;
    private final PasswordEncoder passwordEncoder;
    private final FileService fileService;

    /**
     * 게시글 리스트 조회
     *
     * @param articleSearchRequest 페이지 정보가 담긴 쿼리스트링 DTO
     * @return 페이지에 해당하는 게시글 목록
     */
    public PageResponse<ArticleListItemResponse> getArticleList(ArticleSearchRequest articleSearchRequest) {
        ArticleSearchParams params = ArticleSearchParams.from(articleSearchRequest);
        List<Article> articleList = articleMapper.findAllByParams(params);
        Long totalElements = articleMapper.countByParams(params);

        List<ArticleListItemResponse> articleListResponse = articleList
                .stream()
                .map(ArticleListItemResponse::from)
                .toList();

        return PageResponse.of(params.getPage(), params.getLimit(), totalElements, articleListResponse);
    }

    /**
     * 단일 게시글 조회
     *
     * @param id 단일 게시글의 id
     * @return 게시글 상세 정보가 담긴 DTO
     */
    public ArticleDetailResponse getArticleById(Long id) {
        Article article = getArticleOrThrow(id);
        if (article.isDeleted()) {
            return ArticleDetailResponse.deleted(article);
        }
//        List<FileResponse> files = fileService.findAllByArticleId(id);
        return ArticleDetailResponse.from(article, null);
    }

    /**
     * 게시글 생성
     *
     * @param article 게시글 생성 정보가 담긴 모델 객체
     * @return 생성된 게시글의 id
     */
    public Long createArticle(Article article, List<MultipartFile> files) {
        article.setPassword(passwordEncoder.encode(article.getPassword()));
        articleMapper.insert(article);
        Long articleId = article.getId();
        fileService.uploadArticleFiles(articleId, files);
        return articleId;
    }

    /**
     * 게시글 수정
     *
     * @param id              게시글 id
     * @param modifiedArticle 게시글 수정 요청이 담긴 모델 객체
     */
    public void updateArticle(Long id, Article modifiedArticle) {
        Article article = getArticleOrThrow(id);
        validPasswordOrThrow(modifiedArticle.getPassword(), article.getPassword());
        article.updateInfo(modifiedArticle);
        articleMapper.update(article);
    }

    /**
     * 게시글 삭제 (soft delete)
     *
     * @param id 게시글 id
     */
    public void deleteArticle(Long id, String requestedPassword) {
        Article article = getArticleOrThrow(id);
        validPasswordOrThrow(requestedPassword, article.getPassword());
        articleMapper.delete(id);
    }

    /**
     * 게시글 조회수 증가 (DB에서 처리)
     *
     * @param id 게시글 id
     */
    public void increaseViewCount(Long id) {
        articleMapper.increaseViewCount(id);
    }

    /**
     * 비밀번호 검증 및 불일치시 예외처리
     *
     * @param requestPassword 검증할 비밀번호
     * @param password        변환된 비밀번호
     */
    private void validPasswordOrThrow(String requestPassword, String password) {
        if (!passwordEncoder.matches(requestPassword, password)) {
            throw new BadRequestException("비밀번호가 일치하지 않습니다.");
        }
    }

    /**
     * 게시글 조회 및 실패시 예외처리
     *
     * @param id 게시글 Id
     */
    private Article getArticleOrThrow(Long id) {
        return articleMapper.findById(id).orElseThrow(() -> new IllegalArgumentException("게시글 ID를 찾을 수 없습니다."));
    }

}

```
