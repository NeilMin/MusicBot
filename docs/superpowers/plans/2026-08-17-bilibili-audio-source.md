# Bilibili Audio Source Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let JMusicBot play the audio of Bilibili (哔哩哔哩) videos, via URL or keyword search.

**Architecture:** A native lavaplayer `AudioSourceManager` in a new package
`com.jagrosh.jmusicbot.audio.bilibili`. Metadata is fetched at load time; the expiring
CDN stream URL is resolved lazily at playback time and decoded by lavaplayer's existing
`MpegAudioTrack`. Pure logic (link parsing, WBI signing, JSON parsing) is separated from
I/O so it is testable offline.

**Tech Stack:** Java 25, lavaplayer 2.2.7, Jackson (already a dependency), Apache
HttpClient (arrives with lavaplayer), JUnit 5, Maven.

**Spec:** `docs/superpowers/specs/2026-08-17-bilibili-audio-source-design.md`

## Global Constraints

- **Zero new Maven dependencies.** Jackson, Apache HttpClient, and `java.security.MessageDigest` cover everything.
- **No native libraries and no external binaries.** No yt-dlp, no ffmpeg. Must run from the shaded `JMusicBot-*-All.jar` alone.
- **Must run on Windows, macOS, and ARM64 Linux under Docker.**
- **MD5 must come from `java.security.MessageDigest`** (in `java.base`), so the Docker jlink module list is unchanged.
- **Anonymous access only.** No login, no SESSDATA, no credential storage.
- Java source/target/release is **25**.
- Build command: `mvn -B verify`. Set `JAVA_HOME=/opt/homebrew/opt/openjdk@25/libexec/openjdk.jdk/Contents/Home` first.
- Existing code style: 4-space indent, Allman braces in the `audio` package, Apache 2.0 header on every new file.
- A multi-page (分P) video **always resolves to exactly one track**. Never expand to a playlist.

---

### Task 1: BilibiliLink — input parsing

Pure input parsing. No I/O, so it is fully unit-testable.

**Files:**
- Create: `src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliLink.java`
- Test: `src/test/java/com/jagrosh/jmusicbot/unit/bilibili/BilibiliLinkTest.java`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `enum BilibiliLink.Kind { VIDEO, SEARCH, SHORT_LINK }`
  - `record BilibiliLink(Kind kind, String id, int page)` — `id` is a `BV` id, an `av` id (`"av170001"` form kept as-is), a search query, or a `b23.tv` URL.
  - `static BilibiliLink parse(String input)` — returns `null` when the input is not Bilibili's.
  - `static final String SEARCH_PREFIX = "bilisearch:"`

- [ ] **Step 1: Write the failing test**

```java
package com.jagrosh.jmusicbot.unit.bilibili;

import com.jagrosh.jmusicbot.audio.bilibili.BilibiliLink;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("BilibiliLink Parsing Tests")
class BilibiliLinkTest
{
    @Test
    @DisplayName("parses a standard video URL to page 1")
    void parsesStandardVideoUrl()
    {
        BilibiliLink link = BilibiliLink.parse("https://www.bilibili.com/video/BV1DTbv6xEHK");
        assertNotNull(link);
        assertEquals(BilibiliLink.Kind.VIDEO, link.kind());
        assertEquals("BV1DTbv6xEHK", link.id());
        assertEquals(1, link.page());
    }

    @Test
    @DisplayName("honours the ?p= page parameter")
    void honoursPageParameter()
    {
        BilibiliLink link = BilibiliLink.parse("https://www.bilibili.com/video/BV1GFbk6LEVm?p=3");
        assertEquals(3, link.page());
        assertEquals("BV1GFbk6LEVm", link.id());
    }

    @Test
    @DisplayName("defaults to page 1 when p is absent, zero, or negative")
    void defaultsPageToOne()
    {
        assertEquals(1, BilibiliLink.parse("https://www.bilibili.com/video/BV1GFbk6LEVm?p=0").page());
        assertEquals(1, BilibiliLink.parse("https://www.bilibili.com/video/BV1GFbk6LEVm?p=-2").page());
        assertEquals(1, BilibiliLink.parse("https://www.bilibili.com/video/BV1GFbk6LEVm?p=abc").page());
    }

    @Test
    @DisplayName("parses av ids and mobile/short host variants")
    void parsesAvAndHostVariants()
    {
        assertEquals("av170001", BilibiliLink.parse("https://www.bilibili.com/video/av170001").id());
        assertEquals("BV1DTbv6xEHK", BilibiliLink.parse("https://m.bilibili.com/video/BV1DTbv6xEHK").id());
        assertEquals("BV1DTbv6xEHK", BilibiliLink.parse("bilibili.com/video/BV1DTbv6xEHK").id());
    }

    @Test
    @DisplayName("parses a bare BV id")
    void parsesBareBvId()
    {
        BilibiliLink link = BilibiliLink.parse("BV1DTbv6xEHK");
        assertEquals(BilibiliLink.Kind.VIDEO, link.kind());
        assertEquals("BV1DTbv6xEHK", link.id());
    }

    @Test
    @DisplayName("parses a b23.tv short link as SHORT_LINK")
    void parsesShortLink()
    {
        BilibiliLink link = BilibiliLink.parse("https://b23.tv/AbCdEfG");
        assertEquals(BilibiliLink.Kind.SHORT_LINK, link.kind());
        assertEquals("https://b23.tv/AbCdEfG", link.id());
    }

    @Test
    @DisplayName("parses the bilisearch: prefix and trims the query")
    void parsesSearchPrefix()
    {
        BilibiliLink link = BilibiliLink.parse("bilisearch:  周杰伦 稻香  ");
        assertEquals(BilibiliLink.Kind.SEARCH, link.kind());
        assertEquals("周杰伦 稻香", link.id());
    }

    @Test
    @DisplayName("returns null for non-Bilibili input")
    void returnsNullForForeignInput()
    {
        assertNull(BilibiliLink.parse("https://www.youtube.com/watch?v=dQw4w9WgXcQ"));
        assertNull(BilibiliLink.parse("never gonna give you up"));
        assertNull(BilibiliLink.parse(null));
        assertNull(BilibiliLink.parse(""));
        assertNull(BilibiliLink.parse("bilisearch:   "));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -B test -Dtest=BilibiliLinkTest`
Expected: FAIL — compilation error, `BilibiliLink` does not exist.

- [ ] **Step 3: Write minimal implementation**

```java
package com.jagrosh.jmusicbot.audio.bilibili;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Parses user input into a Bilibili request descriptor.
 *
 * <p>This class performs no I/O, which keeps URL handling fully unit-testable.
 * {@code b23.tv} short links cannot be resolved without a network round trip, so they
 * are reported as {@link Kind#SHORT_LINK} for the source manager to expand.
 */
public record BilibiliLink(Kind kind, String id, int page)
{
    public enum Kind { VIDEO, SEARCH, SHORT_LINK }

    public static final String SEARCH_PREFIX = "bilisearch:";

    private static final Pattern VIDEO_URL = Pattern.compile(
            "^(?:https?://)?(?:www\\.|m\\.)?bilibili\\.com/video/((?:BV[0-9A-Za-z]{10})|(?:[Aa][Vv]\\d+))/?(?:\\?.*)?$");
    private static final Pattern BARE_BV = Pattern.compile("^BV[0-9A-Za-z]{10}$");
    private static final Pattern SHORT_URL = Pattern.compile("^(?:https?://)?b23\\.tv/[0-9A-Za-z]+/?(?:\\?.*)?$");
    private static final Pattern PAGE_PARAM = Pattern.compile("[?&]p=(-?\\d+)");

    public static BilibiliLink parse(String input)
    {
        if(input == null)
            return null;

        String trimmed = input.trim();
        if(trimmed.isEmpty())
            return null;

        if(trimmed.regionMatches(true, 0, SEARCH_PREFIX, 0, SEARCH_PREFIX.length()))
        {
            String query = trimmed.substring(SEARCH_PREFIX.length()).trim();
            return query.isEmpty() ? null : new BilibiliLink(Kind.SEARCH, query, 1);
        }

        if(SHORT_URL.matcher(trimmed).matches())
            return new BilibiliLink(Kind.SHORT_LINK, trimmed, 1);

        Matcher video = VIDEO_URL.matcher(trimmed);
        if(video.matches())
            return new BilibiliLink(Kind.VIDEO, normalizeId(video.group(1)), extractPage(trimmed));

        if(BARE_BV.matcher(trimmed).matches())
            return new BilibiliLink(Kind.VIDEO, trimmed, 1);

        return null;
    }

    /** Lowercases the {@code av} prefix so ids compare equal regardless of input casing. */
    private static String normalizeId(String id)
    {
        return id.regionMatches(true, 0, "av", 0, 2) ? "av" + id.substring(2) : id;
    }

    /** Reads {@code ?p=N}, falling back to 1 for absent, unparseable, or non-positive values. */
    private static int extractPage(String url)
    {
        Matcher matcher = PAGE_PARAM.matcher(url);
        if(!matcher.find())
            return 1;
        try
        {
            int page = Integer.parseInt(matcher.group(1));
            return page > 0 ? page : 1;
        }
        catch(NumberFormatException ignored)
        {
            return 1;
        }
    }
}
```

Prepend the standard Apache 2.0 header used by the other files in `audio/`.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -B test -Dtest=BilibiliLinkTest`
Expected: PASS, 8 tests.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliLink.java \
        src/test/java/com/jagrosh/jmusicbot/unit/bilibili/BilibiliLinkTest.java
git commit -m "feat(bilibili): parse Bilibili links and search queries"
```

---

### Task 2: WbiSigner — WBI request signing

Bilibili's web API requires a `w_rid`/`wts` signature. Pure computation, pinned by real
key material captured from the live API.

**Files:**
- Create: `src/main/java/com/jagrosh/jmusicbot/audio/bilibili/WbiSigner.java`
- Test: `src/test/java/com/jagrosh/jmusicbot/unit/bilibili/WbiSignerTest.java`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `static String mixinKey(String imgKey, String subKey)` — 32-char key.
  - `static String sign(Map<String,String> params, String mixinKey, long wtsSeconds)` — returns the full signed query string, including `wts` and `w_rid`.
  - `static String keyFromUrl(String url)` — extracts `7cd0849…` from `https://…/7cd0849….png`.

- [ ] **Step 1: Write the failing test**

Test vectors below were captured from the live API and cross-checked, so they pin both
the permutation table and the MD5 step.

```java
package com.jagrosh.jmusicbot.unit.bilibili;

import com.jagrosh.jmusicbot.audio.bilibili.WbiSigner;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.util.LinkedHashMap;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("WbiSigner Tests")
class WbiSignerTest
{
    private static final String IMG_KEY = "7cd084941338484aae1ad9425b84077c";
    private static final String SUB_KEY = "4932caff0ff746eab6f01bf08b70ac45";
    private static final String MIXIN_KEY = "ea1db124af3c7062474693fa704f4ff8";

    @Test
    @DisplayName("derives the mixin key from real key material")
    void derivesMixinKey()
    {
        assertEquals(MIXIN_KEY, WbiSigner.mixinKey(IMG_KEY, SUB_KEY));
    }

    @Test
    @DisplayName("mixin key is always 32 characters")
    void mixinKeyLength()
    {
        assertEquals(32, WbiSigner.mixinKey(IMG_KEY, SUB_KEY).length());
    }

    @Test
    @DisplayName("extracts the key from a wbi image URL")
    void extractsKeyFromUrl()
    {
        assertEquals(IMG_KEY,
                WbiSigner.keyFromUrl("https://i0.hdslb.com/bfs/wbi/" + IMG_KEY + ".png"));
    }

    @Test
    @DisplayName("signs playurl parameters to the expected w_rid")
    void signsPlayurlParameters()
    {
        Map<String, String> params = new LinkedHashMap<>();
        params.put("fourk", "1");
        params.put("bvid", "BV1DTbv6xEHK");
        params.put("cid", "40990605342");
        params.put("fnval", "16");
        params.put("fnver", "0");

        String query = WbiSigner.sign(params, MIXIN_KEY, 1755400000L);

        assertEquals("bvid=BV1DTbv6xEHK&cid=40990605342&fnval=16&fnver=0&fourk=1"
                + "&wts=1755400000&w_rid=b2ccf4c580d6c55ccd9dbe3aa170907e", query);
    }

    @Test
    @DisplayName("signs a CJK search query, encoding spaces as plus")
    void signsSearchQuery()
    {
        Map<String, String> params = new LinkedHashMap<>();
        params.put("search_type", "video");
        params.put("keyword", "周杰伦 稻香");
        params.put("page", "1");

        String query = WbiSigner.sign(params, MIXIN_KEY, 1755400000L);

        assertTrue(query.endsWith("&w_rid=6d3769134fbbf27529071cfd988de7fb"), query);
        assertTrue(query.contains("keyword=%E5%91%A8%E6%9D%B0%E4%BC%A6+%E7%A8%BB%E9%A6%99"), query);
    }

    @Test
    @DisplayName("strips the characters Bilibili excludes from signed values")
    void stripsExcludedCharacters()
    {
        Map<String, String> raw = new LinkedHashMap<>();
        raw.put("keyword", "a!b'c(d)e*f");
        Map<String, String> clean = new LinkedHashMap<>();
        clean.put("keyword", "abcdef");

        assertEquals(WbiSigner.sign(clean, MIXIN_KEY, 1755400000L),
                WbiSigner.sign(raw, MIXIN_KEY, 1755400000L));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -B test -Dtest=WbiSignerTest`
Expected: FAIL — compilation error, `WbiSigner` does not exist.

- [ ] **Step 3: Write minimal implementation**

```java
package com.jagrosh.jmusicbot.audio.bilibili;

import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Map;
import java.util.TreeMap;

/**
 * Implements Bilibili's WBI request signing.
 *
 * <p>Signed requests carry a {@code wts} timestamp and a {@code w_rid} MD5 digest. The
 * digest key is derived by permuting the concatenation of two keys published by the
 * {@code nav} endpoint through a fixed table.
 *
 * <p>MD5 comes from {@link MessageDigest}, which lives in {@code java.base}. That keeps
 * the Docker image's jlink module list unchanged.
 */
public final class WbiSigner
{
    /** Bilibili's fixed permutation table for deriving the mixin key. */
    private static final int[] MIXIN_KEY_TABLE = {
        46, 47, 18,  2, 53,  8, 23, 32, 15, 50, 10, 31, 58,  3, 45, 35,
        27, 43,  5, 49, 33,  9, 42, 19, 29, 28, 14, 39, 12, 38, 41, 13,
        37, 48,  7, 16, 24, 55, 40, 61, 26, 17,  0,  1, 60, 51, 30,  4,
        22, 25, 54, 21, 56, 59,  6, 63, 57, 62, 11, 36, 20, 34, 44, 52
    };

    /** Characters Bilibili strips from parameter values before signing. */
    private static final String EXCLUDED_CHARACTERS = "!'()*";

    private static final int MIXIN_KEY_LENGTH = 32;

    private WbiSigner() {}

    public static String mixinKey(String imgKey, String subKey)
    {
        String raw = imgKey + subKey;
        StringBuilder builder = new StringBuilder(MIXIN_KEY_LENGTH);
        for(int index : MIXIN_KEY_TABLE)
        {
            if(builder.length() == MIXIN_KEY_LENGTH)
                break;
            if(index < raw.length())
                builder.append(raw.charAt(index));
        }
        return builder.toString();
    }

    /** Extracts {@code <key>} from a {@code .../<key>.png} URL. */
    public static String keyFromUrl(String url)
    {
        String fileName = url.substring(url.lastIndexOf('/') + 1);
        int dot = fileName.lastIndexOf('.');
        return dot < 0 ? fileName : fileName.substring(0, dot);
    }

    /**
     * Signs the given parameters, returning a complete query string ending in
     * {@code &w_rid=...}. Parameters are sorted by key, as Bilibili requires.
     */
    public static String sign(Map<String, String> params, String mixinKey, long wtsSeconds)
    {
        Map<String, String> sorted = new TreeMap<>(params);
        sorted.put("wts", Long.toString(wtsSeconds));

        StringBuilder query = new StringBuilder();
        for(Map.Entry<String, String> entry : sorted.entrySet())
        {
            if(query.length() > 0)
                query.append('&');
            query.append(encode(entry.getKey()))
                 .append('=')
                 .append(encode(strip(entry.getValue())));
        }

        return query + "&w_rid=" + md5(query + mixinKey);
    }

    private static String strip(String value)
    {
        StringBuilder builder = new StringBuilder(value.length());
        for(int i = 0; i < value.length(); i++)
        {
            char c = value.charAt(i);
            if(EXCLUDED_CHARACTERS.indexOf(c) < 0)
                builder.append(c);
        }
        return builder.toString();
    }

    private static String encode(String value)
    {
        return URLEncoder.encode(value, StandardCharsets.UTF_8);
    }

    private static String md5(String input)
    {
        try
        {
            byte[] digest = MessageDigest.getInstance("MD5")
                    .digest(input.getBytes(StandardCharsets.UTF_8));
            StringBuilder hex = new StringBuilder(digest.length * 2);
            for(byte b : digest)
                hex.append(Character.forDigit((b >> 4) & 0xF, 16))
                   .append(Character.forDigit(b & 0xF, 16));
            return hex.toString();
        }
        catch(NoSuchAlgorithmException e)
        {
            throw new IllegalStateException("MD5 is required but unavailable", e);
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -B test -Dtest=WbiSignerTest`
Expected: PASS, 6 tests.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/jagrosh/jmusicbot/audio/bilibili/WbiSigner.java \
        src/test/java/com/jagrosh/jmusicbot/unit/bilibili/WbiSignerTest.java
git commit -m "feat(bilibili): implement WBI request signing"
```

---

### Task 3: Response models and parser

Turns Bilibili JSON into typed values. Pure functions over `JsonNode`, tested against
real captured responses in `src/test/resources/bilibili/`.

**Files:**
- Create: `src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliVideo.java`
- Create: `src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliAudioStream.java`
- Create: `src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliResponseParser.java`
- Test: `src/test/java/com/jagrosh/jmusicbot/unit/bilibili/BilibiliResponseParserTest.java`

Fixtures already committed: `view-single-page.json`, `view-multi-page.json`,
`playurl-dash.json`, `search-video.json`, `view-error-notfound.json`, `playurl-error.json`.

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces:
  - `record BilibiliVideo(String bvid, long cid, String title, String author, long durationMs, String thumbnailUrl)`
  - `record BilibiliAudioStream(String url, List<String> backupUrls, int bandwidth)`
  - `static BilibiliVideo parseView(JsonNode root, int page)` — selects `pages[page-1]`, falling back to page 1 when out of range.
  - `static BilibiliAudioStream parseAudioStream(JsonNode root)` — highest-bandwidth DASH audio, else the first `durl`.
  - `static List<BilibiliVideo> parseSearch(JsonNode root)` — skips entries with a blank `bvid`.
  - `static void checkCode(JsonNode root)` — throws `FriendlyException` when `code != 0`.

- [ ] **Step 1: Write the failing test**

```java
package com.jagrosh.jmusicbot.unit.bilibili;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.jagrosh.jmusicbot.audio.bilibili.BilibiliAudioStream;
import com.jagrosh.jmusicbot.audio.bilibili.BilibiliResponseParser;
import com.jagrosh.jmusicbot.audio.bilibili.BilibiliVideo;
import com.sedmelluq.discord.lavaplayer.tools.FriendlyException;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.io.InputStream;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("BilibiliResponseParser Tests")
class BilibiliResponseParserTest
{
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private static JsonNode fixture(String name) throws Exception
    {
        try(InputStream in = BilibiliResponseParserTest.class
                .getResourceAsStream("/bilibili/" + name))
        {
            assertNotNull(in, "missing fixture: " + name);
            return MAPPER.readTree(in);
        }
    }

    @Test
    @DisplayName("parses metadata from a single-page video")
    void parsesSinglePageVideo() throws Exception
    {
        BilibiliVideo video = BilibiliResponseParser.parseView(fixture("view-single-page.json"), 1);

        assertFalse(video.bvid().isBlank());
        assertTrue(video.cid() > 0);
        assertFalse(video.title().isBlank());
        assertFalse(video.author().isBlank());
        assertTrue(video.durationMs() > 0, "duration should be in milliseconds");
        assertTrue(video.thumbnailUrl().startsWith("http"));
    }

    @Test
    @DisplayName("selects the requested page of a multi-page video")
    void selectsRequestedPage() throws Exception
    {
        JsonNode root = fixture("view-multi-page.json");
        BilibiliVideo first = BilibiliResponseParser.parseView(root, 1);
        BilibiliVideo third = BilibiliResponseParser.parseView(root, 3);

        assertNotEquals(first.cid(), third.cid(), "each page has its own cid");
        assertNotEquals(first.durationMs(), 0);
    }

    @Test
    @DisplayName("falls back to page 1 when the page is out of range")
    void fallsBackToFirstPage() throws Exception
    {
        JsonNode root = fixture("view-multi-page.json");
        assertEquals(BilibiliResponseParser.parseView(root, 1).cid(),
                     BilibiliResponseParser.parseView(root, 999).cid());
    }

    @Test
    @DisplayName("picks the highest-bandwidth DASH audio stream")
    void picksHighestBandwidthAudio() throws Exception
    {
        BilibiliAudioStream stream =
                BilibiliResponseParser.parseAudioStream(fixture("playurl-dash.json"));

        assertTrue(stream.url().startsWith("http"));
        assertTrue(stream.bandwidth() > 0);
        assertNotNull(stream.backupUrls());
    }

    @Test
    @DisplayName("parses search results and skips entries without a bvid")
    void parsesSearchResults() throws Exception
    {
        List<BilibiliVideo> results = BilibiliResponseParser.parseSearch(fixture("search-video.json"));

        assertFalse(results.isEmpty());
        assertTrue(results.stream().noneMatch(v -> v.bvid() == null || v.bvid().isBlank()),
                "ad/placeholder entries must be filtered out");
        assertTrue(results.stream().allMatch(v -> !v.title().contains("<em")),
                "search titles must have their HTML highlight tags stripped");
    }

    @Test
    @DisplayName("throws a FriendlyException carrying Bilibili's message on an error code")
    void throwsOnErrorCode() throws Exception
    {
        JsonNode root = fixture("view-error-notfound.json");
        FriendlyException thrown = assertThrows(FriendlyException.class,
                () -> BilibiliResponseParser.checkCode(root));
        assertTrue(thrown.getMessage().contains("啥都木有"), thrown.getMessage());
    }

    @Test
    @DisplayName("accepts a successful response")
    void acceptsSuccess() throws Exception
    {
        assertDoesNotThrow(() -> BilibiliResponseParser.checkCode(fixture("view-single-page.json")));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -B test -Dtest=BilibiliResponseParserTest`
Expected: FAIL — compilation error, the parser classes do not exist.

- [ ] **Step 3: Write minimal implementation**

`BilibiliVideo.java`:

```java
package com.jagrosh.jmusicbot.audio.bilibili;

/** Metadata for one playable Bilibili video page. */
public record BilibiliVideo(String bvid, long cid, String title, String author,
                            long durationMs, String thumbnailUrl)
{
}
```

`BilibiliAudioStream.java`:

```java
package com.jagrosh.jmusicbot.audio.bilibili;

import java.util.List;

/**
 * A resolved audio stream. The URLs expire roughly 120 minutes after they are issued,
 * so instances are short-lived and resolved at playback time.
 */
public record BilibiliAudioStream(String url, List<String> backupUrls, int bandwidth)
{
}
```

`BilibiliResponseParser.java`:

```java
package com.jagrosh.jmusicbot.audio.bilibili;

import java.util.ArrayList;
import java.util.List;

import com.fasterxml.jackson.databind.JsonNode;
import com.sedmelluq.discord.lavaplayer.tools.FriendlyException;

/**
 * Converts Bilibili API responses into typed values.
 *
 * <p>Every method is a pure function of its {@link JsonNode} input, so the whole class is
 * tested offline against captured real responses.
 */
public final class BilibiliResponseParser
{
    private BilibiliResponseParser() {}

    /** Throws when the response carries a non-zero {@code code}, preserving Bilibili's message. */
    public static void checkCode(JsonNode root)
    {
        int code = root.path("code").asInt(-1);
        if(code == 0)
            return;

        String message = root.path("message").asText("Unknown Bilibili error");
        throw new FriendlyException("Bilibili rejected the request: " + message + " (code " + code + ")",
                FriendlyException.Severity.COMMON, null);
    }

    /**
     * Reads video metadata, selecting the given 1-based page. Out-of-range pages fall back
     * to page 1 rather than failing, since a bad {@code ?p=} should not block playback.
     */
    public static BilibiliVideo parseView(JsonNode root, int page)
    {
        checkCode(root);
        JsonNode data = root.path("data");

        JsonNode pages = data.path("pages");
        int index = (page >= 1 && page <= pages.size()) ? page - 1 : 0;
        JsonNode selected = pages.isArray() && !pages.isEmpty() ? pages.get(index) : null;

        long cid = selected != null ? selected.path("cid").asLong() : data.path("cid").asLong();
        long seconds = selected != null ? selected.path("duration").asLong() : data.path("duration").asLong();

        String title = data.path("title").asText();
        if(selected != null && pages.size() > 1)
        {
            String pageTitle = selected.path("part").asText("");
            if(!pageTitle.isBlank() && !pageTitle.equals(title))
                title = title + " - P" + (index + 1) + " " + pageTitle;
        }

        return new BilibiliVideo(
                data.path("bvid").asText(),
                cid,
                title,
                data.path("owner").path("name").asText(),
                seconds * 1000L,
                data.path("pic").asText());
    }

    /**
     * Selects the best audio stream: the highest-bandwidth DASH entry when present,
     * otherwise the progressive {@code durl} that older videos still return.
     */
    public static BilibiliAudioStream parseAudioStream(JsonNode root)
    {
        checkCode(root);
        JsonNode data = root.path("data");

        JsonNode audioList = data.path("dash").path("audio");
        if(audioList.isArray() && !audioList.isEmpty())
        {
            JsonNode best = null;
            for(JsonNode candidate : audioList)
                if(best == null || candidate.path("bandwidth").asInt() > best.path("bandwidth").asInt())
                    best = candidate;

            List<String> backups = new ArrayList<>();
            for(JsonNode backup : best.path("backupUrl"))
                backups.add(backup.asText());

            return new BilibiliAudioStream(best.path("baseUrl").asText(), backups,
                    best.path("bandwidth").asInt());
        }

        JsonNode durl = data.path("durl");
        if(durl.isArray() && !durl.isEmpty())
        {
            List<String> backups = new ArrayList<>();
            for(JsonNode backup : durl.get(0).path("backup_url"))
                backups.add(backup.asText());
            return new BilibiliAudioStream(durl.get(0).path("url").asText(), backups, 0);
        }

        throw new FriendlyException("Bilibili returned no playable audio stream for this video",
                FriendlyException.Severity.SUSPICIOUS, null);
    }

    /** Reads search results, skipping ad and placeholder entries that carry no {@code bvid}. */
    public static List<BilibiliVideo> parseSearch(JsonNode root)
    {
        checkCode(root);
        List<BilibiliVideo> videos = new ArrayList<>();

        for(JsonNode item : root.path("data").path("result"))
        {
            String bvid = item.path("bvid").asText("");
            if(bvid.isBlank())
                continue;

            videos.add(new BilibiliVideo(
                    bvid,
                    0L,
                    stripHighlight(item.path("title").asText()),
                    item.path("author").asText(),
                    parseDuration(item.path("duration").asText("")),
                    normalizeThumbnail(item.path("pic").asText(""))));
        }
        return videos;
    }

    /** Search titles wrap matched terms in {@code <em>} tags; the queue should show plain text. */
    private static String stripHighlight(String title)
    {
        return title.replaceAll("<[^>]+>", "");
    }

    /** Search results express duration as {@code m:ss} or {@code h:mm:ss}, not seconds. */
    private static long parseDuration(String duration)
    {
        if(duration.isBlank())
            return 0L;

        long total = 0L;
        for(String part : duration.split(":"))
        {
            try
            {
                total = total * 60 + Long.parseLong(part.trim());
            }
            catch(NumberFormatException e)
            {
                return 0L;
            }
        }
        return total * 1000L;
    }

    /** Search thumbnails come back protocol-relative ({@code //i0.hdslb.com/...}). */
    private static String normalizeThumbnail(String pic)
    {
        return pic.startsWith("//") ? "https:" + pic : pic;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -B test -Dtest=BilibiliResponseParserTest`
Expected: PASS, 7 tests.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/jagrosh/jmusicbot/audio/bilibili/ \
        src/test/java/com/jagrosh/jmusicbot/unit/bilibili/BilibiliResponseParserTest.java
git commit -m "feat(bilibili): parse view, playurl, and search responses"
```

---

### Task 4: BilibiliApiClient — HTTP access

The only class that performs network I/O against Bilibili. Thin by design: it builds
requests, reads JSON, and hands off to `BilibiliResponseParser`.

**Files:**
- Create: `src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliApiClient.java`
- Test: `src/test/java/com/jagrosh/jmusicbot/unit/bilibili/BilibiliApiClientTest.java`

**Interfaces:**
- Consumes: `WbiSigner`, `BilibiliResponseParser`, `BilibiliVideo`, `BilibiliAudioStream`.
- Produces:
  - `BilibiliApiClient()` constructor.
  - `BilibiliVideo loadVideo(HttpInterface http, String id, int page)`
  - `BilibiliAudioStream loadAudioStream(HttpInterface http, String bvid, long cid)`
  - `List<BilibiliVideo> search(HttpInterface http, String query, int limit)`
  - `String resolveShortLink(HttpInterface http, String shortUrl)`
  - `static final String USER_AGENT`, `static final String REFERER` — the two headers Bilibili's CDN requires.

Cache the WBI mixin key for 12 hours; refresh on expiry. If the `nav` call fails, fall
back to the unsigned `/x/player/playurl` endpoint, which is verified to work.

- [ ] **Step 1: Write the failing test**

Network-dependent behavior is covered by the opt-in integration test in Task 8. These
tests pin the offline-checkable contract.

```java
package com.jagrosh.jmusicbot.unit.bilibili;

import com.jagrosh.jmusicbot.audio.bilibili.BilibiliApiClient;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("BilibiliApiClient Tests")
class BilibiliApiClientTest
{
    @Test
    @DisplayName("sends a Referer, which Bilibili's CDN requires")
    void definesReferer()
    {
        assertEquals("https://www.bilibili.com", BilibiliApiClient.REFERER);
    }

    @Test
    @DisplayName("sends a browser User-Agent, which Bilibili's CDN requires")
    void definesBrowserUserAgent()
    {
        assertTrue(BilibiliApiClient.USER_AGENT.contains("Mozilla/5.0"), BilibiliApiClient.USER_AGENT);
    }

    @Test
    @DisplayName("builds a signed playurl query containing the required parameters")
    void buildsSignedPlayurlQuery()
    {
        String query = BilibiliApiClient.buildPlayurlQuery("BV1DTbv6xEHK", 40990605342L,
                "ea1db124af3c7062474693fa704f4ff8", 1755400000L);

        assertTrue(query.contains("bvid=BV1DTbv6xEHK"), query);
        assertTrue(query.contains("cid=40990605342"), query);
        assertTrue(query.contains("fnval=16"), query);
        assertTrue(query.contains("w_rid="), query);
        assertTrue(query.contains("wts=1755400000"), query);
    }

    @Test
    @DisplayName("uses the avid parameter for av ids and bvid for BV ids")
    void selectsIdParameter()
    {
        assertTrue(BilibiliApiClient.buildViewQuery("av170001").contains("aid=170001"));
        assertTrue(BilibiliApiClient.buildViewQuery("BV1DTbv6xEHK").contains("bvid=BV1DTbv6xEHK"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -B test -Dtest=BilibiliApiClientTest`
Expected: FAIL — compilation error, `BilibiliApiClient` does not exist.

- [ ] **Step 3: Write minimal implementation**

```java
package com.jagrosh.jmusicbot.audio.bilibili;

import java.io.IOException;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sedmelluq.discord.lavaplayer.tools.FriendlyException;
import com.sedmelluq.discord.lavaplayer.tools.io.HttpInterface;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Performs the HTTP calls against Bilibili's web API.
 *
 * <p>Bilibili's CDN rejects requests that lack a {@code Referer} or that do not present a
 * browser {@code User-Agent}; both return HTTP 403. Those headers are therefore part of
 * this class's contract rather than an optional nicety.
 */
public class BilibiliApiClient
{
    public static final String REFERER = "https://www.bilibili.com";
    public static final String USER_AGENT =
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
            + "(KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36";

    private static final Logger LOGGER = LoggerFactory.getLogger(BilibiliApiClient.class);
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private static final String VIEW_URL = "https://api.bilibili.com/x/web-interface/view?";
    private static final String PLAYURL_SIGNED = "https://api.bilibili.com/x/player/wbi/playurl?";
    private static final String PLAYURL_PLAIN = "https://api.bilibili.com/x/player/playurl?";
    private static final String SEARCH_URL = "https://api.bilibili.com/x/web-interface/wbi/search/type?";
    private static final String NAV_URL = "https://api.bilibili.com/x/web-interface/nav";

    private static final long KEY_TTL_MS = 12 * 60 * 60 * 1000L;

    private volatile String cachedMixinKey;
    private volatile long cachedMixinKeyAt;

    public BilibiliVideo loadVideo(HttpInterface http, String id, int page) throws IOException
    {
        return BilibiliResponseParser.parseView(getJson(http, VIEW_URL + buildViewQuery(id)), page);
    }

    /**
     * Resolves a playable audio stream. Prefers the WBI-signed endpoint; if the signing key
     * cannot be obtained, falls back to the unsigned endpoint so playback degrades rather
     * than breaks.
     */
    public BilibiliAudioStream loadAudioStream(HttpInterface http, String bvid, long cid) throws IOException
    {
        String mixinKey = mixinKey(http);
        long wts = System.currentTimeMillis() / 1000L;

        String url = mixinKey != null
                ? PLAYURL_SIGNED + buildPlayurlQuery(bvid, cid, mixinKey, wts)
                : PLAYURL_PLAIN + "bvid=" + encode(bvid) + "&cid=" + cid + "&fnval=16&fnver=0&fourk=1";

        return BilibiliResponseParser.parseAudioStream(getJson(http, url));
    }

    public List<BilibiliVideo> search(HttpInterface http, String query, int limit) throws IOException
    {
        String mixinKey = mixinKey(http);
        if(mixinKey == null)
            throw new FriendlyException("Bilibili search is unavailable: could not obtain a signing key",
                    FriendlyException.Severity.SUSPICIOUS, null);

        Map<String, String> params = new LinkedHashMap<>();
        params.put("search_type", "video");
        params.put("keyword", query);
        params.put("page", "1");
        params.put("page_size", Integer.toString(limit));

        String signed = WbiSigner.sign(params, mixinKey, System.currentTimeMillis() / 1000L);
        return BilibiliResponseParser.parseSearch(getJson(http, SEARCH_URL + signed));
    }

    /** Follows a {@code b23.tv} redirect to the real video URL. */
    public String resolveShortLink(HttpInterface http, String shortUrl) throws IOException
    {
        HttpGet request = new HttpGet(shortUrl);
        applyHeaders(request);
        try(CloseableHttpResponse response = http.execute(request))
        {
            org.apache.http.util.EntityUtils.consumeQuietly(response.getEntity());
            return http.getFinalLocation() != null ? http.getFinalLocation().toString() : shortUrl;
        }
    }

    static String buildViewQuery(String id)
    {
        if(id.regionMatches(true, 0, "av", 0, 2))
            return "aid=" + encode(id.substring(2));
        return "bvid=" + encode(id);
    }

    static String buildPlayurlQuery(String bvid, long cid, String mixinKey, long wtsSeconds)
    {
        Map<String, String> params = new LinkedHashMap<>();
        params.put("bvid", bvid);
        params.put("cid", Long.toString(cid));
        params.put("fnval", "16");
        params.put("fnver", "0");
        params.put("fourk", "1");
        return WbiSigner.sign(params, mixinKey, wtsSeconds);
    }

    /** Returns the cached mixin key, refreshing it when stale, or null if unavailable. */
    private String mixinKey(HttpInterface http)
    {
        long now = System.currentTimeMillis();
        String cached = cachedMixinKey;
        if(cached != null && now - cachedMixinKeyAt < KEY_TTL_MS)
            return cached;

        try
        {
            JsonNode wbi = getJson(http, NAV_URL).path("data").path("wbi_img");
            String key = WbiSigner.mixinKey(
                    WbiSigner.keyFromUrl(wbi.path("img_url").asText()),
                    WbiSigner.keyFromUrl(wbi.path("sub_url").asText()));
            cachedMixinKey = key;
            cachedMixinKeyAt = now;
            return key;
        }
        catch(Exception e)
        {
            LOGGER.warn("Could not fetch Bilibili WBI keys, falling back to unsigned requests: {}",
                    e.getMessage());
            return null;
        }
    }

    private JsonNode getJson(HttpInterface http, String url) throws IOException
    {
        HttpGet request = new HttpGet(url);
        applyHeaders(request);
        try(CloseableHttpResponse response = http.execute(request))
        {
            int status = response.getStatusLine().getStatusCode();
            if(status != 200)
                throw new IOException("Bilibili API returned HTTP " + status + " for " + url);
            return MAPPER.readTree(response.getEntity().getContent());
        }
    }

    private static void applyHeaders(HttpGet request)
    {
        request.setHeader("Referer", REFERER);
        request.setHeader("User-Agent", USER_AGENT);
    }

    private static String encode(String value)
    {
        return URLEncoder.encode(value, StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -B test -Dtest=BilibiliApiClientTest`
Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliApiClient.java \
        src/test/java/com/jagrosh/jmusicbot/unit/bilibili/BilibiliApiClientTest.java
git commit -m "feat(bilibili): add API client with required CDN headers"
```

---

### Task 5: Source manager and track

Wires everything into lavaplayer. The track resolves its stream at playback time, because
`baseUrl` expires in about 120 minutes and a track can sit in a queue far longer.

**Files:**
- Create: `src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliAudioTrack.java`
- Create: `src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliAudioSourceManager.java`
- Test: `src/test/java/com/jagrosh/jmusicbot/unit/bilibili/BilibiliAudioSourceManagerTest.java`

**Interfaces:**
- Consumes: `BilibiliLink`, `BilibiliApiClient`, `BilibiliVideo`, `BilibiliAudioStream`.
- Produces:
  - `BilibiliAudioSourceManager()` — implements `AudioSourceManager`, `getSourceName()` returns `"bilibili"`.
  - `HttpInterface getHttpInterface()`
  - `BilibiliApiClient getApiClient()`
  - `BilibiliAudioTrack(AudioTrackInfo info, long cid, BilibiliAudioSourceManager manager)`

`encodeTrack`/`decodeTrack` persist the `cid`, so queued tracks survive a restart.

- [ ] **Step 1: Write the failing test**

```java
package com.jagrosh.jmusicbot.unit.bilibili;

import com.jagrosh.jmusicbot.audio.bilibili.BilibiliAudioSourceManager;
import com.sedmelluq.discord.lavaplayer.track.AudioReference;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@DisplayName("BilibiliAudioSourceManager Tests")
class BilibiliAudioSourceManagerTest
{
    private BilibiliAudioSourceManager manager;

    @BeforeEach
    void setUp()
    {
        manager = new BilibiliAudioSourceManager();
    }

    @AfterEach
    void tearDown()
    {
        manager.shutdown();
    }

    @Test
    @DisplayName("reports the source name used in config and logs")
    void reportsSourceName()
    {
        assertEquals("bilibili", manager.getSourceName());
    }

    @Test
    @DisplayName("declines identifiers belonging to other sources without any network call")
    void declinesForeignIdentifiers()
    {
        assertNull(manager.loadItem(null, new AudioReference("https://youtube.com/watch?v=abc", null)));
        assertNull(manager.loadItem(null, new AudioReference("ytsearch:hello", null)));
        assertNull(manager.loadItem(null, new AudioReference("some random text", null)));
    }

    @Test
    @DisplayName("exposes an HTTP interface for the track to stream through")
    void exposesHttpInterface()
    {
        assertNotNull(manager.getHttpInterface());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -B test -Dtest=BilibiliAudioSourceManagerTest`
Expected: FAIL — compilation error, the source manager does not exist.

- [ ] **Step 3: Write minimal implementation**

`BilibiliAudioTrack.java`:

```java
package com.jagrosh.jmusicbot.audio.bilibili;

import java.net.URI;

import com.sedmelluq.discord.lavaplayer.container.mpeg.MpegAudioTrack;
import com.sedmelluq.discord.lavaplayer.source.AudioSourceManager;
import com.sedmelluq.discord.lavaplayer.tools.io.HttpInterface;
import com.sedmelluq.discord.lavaplayer.tools.io.PersistentHttpStream;
import com.sedmelluq.discord.lavaplayer.track.AudioTrack;
import com.sedmelluq.discord.lavaplayer.track.AudioTrackInfo;
import com.sedmelluq.discord.lavaplayer.track.DelegatedAudioTrack;
import com.sedmelluq.discord.lavaplayer.track.playback.LocalAudioTrackExecutor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * A single Bilibili video page, played as audio.
 *
 * <p>The stream URL is resolved here rather than at load time because Bilibili's CDN URLs
 * expire after roughly 120 minutes. A track queued behind a long playlist would otherwise
 * hold a dead URL by the time it started playing.
 */
public class BilibiliAudioTrack extends DelegatedAudioTrack
{
    private static final Logger LOGGER = LoggerFactory.getLogger(BilibiliAudioTrack.class);

    private final long cid;
    private final BilibiliAudioSourceManager sourceManager;

    public BilibiliAudioTrack(AudioTrackInfo trackInfo, long cid, BilibiliAudioSourceManager sourceManager)
    {
        super(trackInfo);
        this.cid = cid;
        this.sourceManager = sourceManager;
    }

    public long getCid()
    {
        return cid;
    }

    @Override
    public void process(LocalAudioTrackExecutor executor) throws Exception
    {
        try(HttpInterface httpInterface = sourceManager.getHttpInterface())
        {
            BilibiliAudioStream stream = sourceManager.getApiClient()
                    .loadAudioStream(httpInterface, trackInfo.identifier, cid);

            LOGGER.debug("Streaming Bilibili {} at {} bps", trackInfo.identifier, stream.bandwidth());

            try(PersistentHttpStream inputStream =
                        new PersistentHttpStream(httpInterface, new URI(stream.url()), null))
            {
                processDelegate(new MpegAudioTrack(trackInfo, inputStream), executor);
            }
        }
    }

    @Override
    protected AudioTrack makeShallowClone()
    {
        return new BilibiliAudioTrack(trackInfo, cid, sourceManager);
    }

    @Override
    public AudioSourceManager getSourceManager()
    {
        return sourceManager;
    }
}
```

`BilibiliAudioSourceManager.java`:

```java
package com.jagrosh.jmusicbot.audio.bilibili;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import com.sedmelluq.discord.lavaplayer.player.AudioPlayerManager;
import com.sedmelluq.discord.lavaplayer.source.AudioSourceManager;
import com.sedmelluq.discord.lavaplayer.tools.ExceptionTools;
import com.sedmelluq.discord.lavaplayer.tools.FriendlyException;
import com.sedmelluq.discord.lavaplayer.tools.io.HttpClientTools;
import com.sedmelluq.discord.lavaplayer.tools.io.HttpInterface;
import com.sedmelluq.discord.lavaplayer.tools.io.HttpInterfaceManager;
import com.sedmelluq.discord.lavaplayer.track.AudioItem;
import com.sedmelluq.discord.lavaplayer.track.AudioReference;
import com.sedmelluq.discord.lavaplayer.track.AudioTrack;
import com.sedmelluq.discord.lavaplayer.track.AudioTrackInfo;
import com.sedmelluq.discord.lavaplayer.track.BasicAudioPlaylist;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.message.BasicHeader;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Plays audio from Bilibili (哔哩哔哩) videos.
 *
 * <p>Accepts video URLs, bare {@code BV} ids, {@code b23.tv} short links, and
 * {@code bilisearch:} queries. A multi-page (分P) video always resolves to exactly one
 * track: the page named by {@code ?p=}, or page 1.
 */
public class BilibiliAudioSourceManager implements AudioSourceManager
{
    private static final Logger LOGGER = LoggerFactory.getLogger(BilibiliAudioSourceManager.class);
    private static final int SEARCH_LIMIT = 20;

    private final HttpInterfaceManager httpInterfaceManager;
    private final BilibiliApiClient apiClient;

    public BilibiliAudioSourceManager()
    {
        this.apiClient = new BilibiliApiClient();
        this.httpInterfaceManager = HttpClientTools.createDefaultThreadLocalManager();
        this.httpInterfaceManager.configureBuilder(this::configureHeaders);
    }

    /**
     * Bilibili's CDN answers 403 unless both a Referer and a browser User-Agent are
     * present, so they are set as defaults on every request this manager makes.
     */
    private void configureHeaders(HttpClientBuilder builder)
    {
        List<BasicHeader> headers = new ArrayList<>();
        headers.add(new BasicHeader("Referer", BilibiliApiClient.REFERER));
        headers.add(new BasicHeader("User-Agent", BilibiliApiClient.USER_AGENT));
        builder.setDefaultHeaders(headers);
    }

    @Override
    public String getSourceName()
    {
        return "bilibili";
    }

    public HttpInterface getHttpInterface()
    {
        return httpInterfaceManager.getInterface();
    }

    public BilibiliApiClient getApiClient()
    {
        return apiClient;
    }

    @Override
    public AudioItem loadItem(AudioPlayerManager manager, AudioReference reference)
    {
        BilibiliLink link = BilibiliLink.parse(reference.identifier);
        if(link == null)
            return null;

        try(HttpInterface httpInterface = getHttpInterface())
        {
            return switch(link.kind())
            {
                case SEARCH -> loadSearch(httpInterface, link.id());
                case VIDEO -> loadVideo(httpInterface, link.id(), link.page());
                case SHORT_LINK -> loadShortLink(httpInterface, link.id());
            };
        }
        catch(FriendlyException e)
        {
            throw e;
        }
        catch(Exception e)
        {
            throw ExceptionTools.wrapUnfriendlyExceptions(
                    "Could not load this Bilibili item.", FriendlyException.Severity.FAULT, e);
        }
    }

    private AudioItem loadVideo(HttpInterface httpInterface, String id, int page) throws IOException
    {
        BilibiliVideo video = apiClient.loadVideo(httpInterface, id, page);
        return buildTrack(video);
    }

    /** Expands a b23.tv link, then loads whatever it points at. Never recurses further. */
    private AudioItem loadShortLink(HttpInterface httpInterface, String shortUrl) throws IOException
    {
        String resolved = apiClient.resolveShortLink(httpInterface, shortUrl);
        BilibiliLink link = BilibiliLink.parse(resolved);

        if(link == null || link.kind() != BilibiliLink.Kind.VIDEO)
        {
            LOGGER.debug("b23.tv link {} resolved to unsupported target {}", shortUrl, resolved);
            return AudioReference.NO_TRACK;
        }
        return loadVideo(httpInterface, link.id(), link.page());
    }

    private AudioItem loadSearch(HttpInterface httpInterface, String query) throws IOException
    {
        List<BilibiliVideo> results = apiClient.search(httpInterface, query, SEARCH_LIMIT);
        if(results.isEmpty())
            return AudioReference.NO_TRACK;

        List<AudioTrack> tracks = new ArrayList<>(results.size());
        for(BilibiliVideo video : results)
            tracks.add(buildTrack(video));

        return new BasicAudioPlaylist("Search results for: " + query, tracks, null, true);
    }

    /**
     * Search results carry no cid, so tracks built from them resolve it at playback time
     * by their bvid. A cid of 0 means "look it up later".
     */
    private AudioTrack buildTrack(BilibiliVideo video)
    {
        AudioTrackInfo info = new AudioTrackInfo(
                video.title(),
                video.author(),
                video.durationMs(),
                video.bvid(),
                false,
                "https://www.bilibili.com/video/" + video.bvid(),
                video.thumbnailUrl(),
                null);
        return new BilibiliAudioTrack(info, video.cid(), this);
    }

    @Override
    public boolean isTrackEncodable(AudioTrack track)
    {
        return true;
    }

    @Override
    public void encodeTrack(AudioTrack track, DataOutput output) throws IOException
    {
        output.writeLong(((BilibiliAudioTrack) track).getCid());
    }

    @Override
    public AudioTrack decodeTrack(AudioTrackInfo trackInfo, DataInput input) throws IOException
    {
        return new BilibiliAudioTrack(trackInfo, input.readLong(), this);
    }

    @Override
    public void shutdown()
    {
        try
        {
            httpInterfaceManager.close();
        }
        catch(IOException e)
        {
            LOGGER.debug("Failed to close Bilibili HTTP interface manager", e);
        }
    }
}
```

Both lavaplayer signatures used above are confirmed present in 2.2.7:
`HttpClientTools.createDefaultThreadLocalManager()` returns `HttpInterfaceManager`, and
`ExceptionTools.wrapUnfriendlyExceptions(String, Severity, Throwable)` returns
`FriendlyException`.

A track built from a search result has `cid == 0`. Handle that in
`BilibiliApiClient.loadAudioStream`: when `cid <= 0`, call `loadVideo` first to obtain
the real cid. Add that guard now.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -B test -Dtest=BilibiliAudioSourceManagerTest`
Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliAudioTrack.java \
        src/main/java/com/jagrosh/jmusicbot/audio/bilibili/BilibiliAudioSourceManager.java \
        src/test/java/com/jagrosh/jmusicbot/unit/bilibili/BilibiliAudioSourceManagerTest.java
git commit -m "feat(bilibili): add source manager with lazy stream resolution"
```

---

### Task 6: Register the source and expose the config toggle

**Files:**
- Modify: `src/main/java/com/jagrosh/jmusicbot/audio/AudioSource.java` (add the enum constant after `NICO`)
- Modify: `src/main/resources/reference.conf` (the `playback.audioSources` block)
- Modify: `src/test/java/com/jagrosh/jmusicbot/unit/AudioSourceUnitTest.java`

**Interfaces:**
- Consumes: `BilibiliAudioSourceManager`.
- Produces: `AudioSource.BILIBILI`, config key `playback.audioSources.bilibili`.

- [ ] **Step 1: Write the failing test**

Add to `AudioSourceUnitTest`:

```java
    @Test
    @DisplayName("BILIBILI is registered as a platform-specific source")
    void bilibiliIsPlatformSpecific()
    {
        assertEquals("bilibili", AudioSource.BILIBILI.getConfigName());
        assertTrue(AudioSource.BILIBILI.getRegistrationPriority() < AudioSource.HTTP.getRegistrationPriority(),
                "Bilibili must be registered before the HTTP catch-all, or HTTP will claim its URLs");
        assertEquals(Optional.of(AudioSource.BILIBILI), AudioSource.fromConfigName("BiliBili"));
    }
```

Also add `AudioSource.BILIBILI` to the `platformSources` array in
`platformSpecificSourcesHaveLowerPriorityThanCatchAll`.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -B test -Dtest=AudioSourceUnitTest`
Expected: FAIL — `AudioSource.BILIBILI` does not exist.

- [ ] **Step 3: Write minimal implementation**

In `AudioSource.java`, add after the `NICO` constant (keeping the trailing comma):

```java
    BILIBILI(
        "bilibili",
        "Bilibili videos and search",
        25,
        (manager, config) -> manager.registerSourceManager(new BilibiliAudioSourceManager())
    ),
```

Add the import:

```java
import com.jagrosh.jmusicbot.audio.bilibili.BilibiliAudioSourceManager;
```

In `reference.conf`, inside `audioSources`, add after `nico = true`:

```
    bilibili = true
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -B test -Dtest=AudioSourceUnitTest`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/com/jagrosh/jmusicbot/audio/AudioSource.java \
        src/main/resources/reference.conf \
        src/test/java/com/jagrosh/jmusicbot/unit/AudioSourceUnitTest.java
git commit -m "feat(bilibili): register the source and add its config toggle"
```

---

### Task 7: Search commands

Mirrors `SCSearchCmd`, which is a small subclass of `SearchCmd` that only sets a prefix.

**Files:**
- Create: `src/main/java/com/jagrosh/jmusicbot/commands/v1/music/BiliSearchCmd.java`
- Modify: `src/main/java/com/jagrosh/jmusicbot/commands/v1/CommandFactory.java` (register alongside `SCSearchCmd`)
- Modify: `src/main/resources/reference.conf` if the aliases block enumerates command names

**Interfaces:**
- Consumes: `BilibiliLink.SEARCH_PREFIX`.
- Produces: the `bilisearch` text command.

- [ ] **Step 1: Read how SCSearchCmd is registered**

Run: `grep -rn "SCSearchCmd" src/main/java/`
Register `BiliSearchCmd` in exactly the same places.

- [ ] **Step 2: Write minimal implementation**

```java
package com.jagrosh.jmusicbot.commands.v1.music;

import com.jagrosh.jmusicbot.Bot;

/**
 * Searches Bilibili for a query and offers the results for selection.
 */
public class BiliSearchCmd extends SearchCmd
{
    public BiliSearchCmd(Bot bot)
    {
        super(bot);
        this.searchPrefix = "bilisearch:";
        this.name = "bilisearch";
        this.help = "searches Bilibili for a provided query";
        this.aliases = bot.getConfig().getAliases(this.name);
    }
}
```

- [ ] **Step 3: Run the full suite**

Run: `mvn -B verify`
Expected: PASS, with the bot still starting up in existing tests.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/com/jagrosh/jmusicbot/commands/v1/music/BiliSearchCmd.java \
        src/main/java/com/jagrosh/jmusicbot/commands/v1/CommandFactory.java
git commit -m "feat(bilibili): add the bilisearch command"
```

---

### Task 8: Live integration test and documentation

**Files:**
- Create: `src/test/java/com/jagrosh/jmusicbot/integration/BilibiliLiveIntegrationTest.java`
- Modify: `README.md`

**Interfaces:**
- Consumes: everything above.
- Produces: no production interfaces.

- [ ] **Step 1: Write the opt-in integration test**

Disabled unless `BILIBILI_LIVE_TEST=1`, so CI stays offline and deterministic.

```java
package com.jagrosh.jmusicbot.integration;

import com.jagrosh.jmusicbot.audio.bilibili.*;
import com.sedmelluq.discord.lavaplayer.tools.io.HttpInterface;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIfEnvironmentVariable;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@EnabledIfEnvironmentVariable(named = "BILIBILI_LIVE_TEST", matches = "1")
@DisplayName("Bilibili Live API Integration")
class BilibiliLiveIntegrationTest
{
    @Test
    @DisplayName("resolves a real video to a playable audio stream")
    void resolvesRealVideo() throws Exception
    {
        BilibiliAudioSourceManager manager = new BilibiliAudioSourceManager();
        try(HttpInterface http = manager.getHttpInterface())
        {
            BilibiliApiClient client = manager.getApiClient();
            List<BilibiliVideo> results = client.search(http, "周杰伦", 5);
            assertFalse(results.isEmpty());

            BilibiliVideo video = client.loadVideo(http, results.get(0).bvid(), 1);
            assertTrue(video.durationMs() > 0);

            BilibiliAudioStream stream = client.loadAudioStream(http, video.bvid(), video.cid());
            assertTrue(stream.url().startsWith("http"));
        }
        finally
        {
            manager.shutdown();
        }
    }
}
```

- [ ] **Step 2: Run it against the live API once**

Run: `BILIBILI_LIVE_TEST=1 mvn -B test -Dtest=BilibiliLiveIntegrationTest`
Expected: PASS.

- [ ] **Step 3: Confirm it is skipped by default**

Run: `mvn -B test -Dtest=BilibiliLiveIntegrationTest`
Expected: skipped, not failed.

- [ ] **Step 4: Document the feature in README.md**

Add Bilibili to the supported-sources list, and document the `bilisearch` command and the
`bilibili` config toggle. State that a multi-page video plays the page given by `?p=`,
or page 1.

- [ ] **Step 5: Verify the shaded jar**

Run: `mvn -B clean package`
Then confirm the classes shipped and that no new dependency was introduced:

```bash
unzip -l target/JMusicBot-*-All.jar | grep -c "audio/bilibili/"
git diff master --stat -- pom.xml
```

Expected: a non-zero class count, and **no change to `pom.xml`**.

- [ ] **Step 6: Commit**

```bash
git add src/test/java/com/jagrosh/jmusicbot/integration/BilibiliLiveIntegrationTest.java README.md
git commit -m "test(bilibili): add opt-in live API test and document the source"
```

---

## Self-Review

**Spec coverage.** Every spec section maps to a task: architecture → Tasks 1–5; input
handling → Task 1 (parsing) and Task 5 (dispatch); playback path → Task 5; registration
and configuration → Task 6; search → Tasks 4 and 7; error handling → Tasks 3 and 4;
testing → every task, plus Task 8. The zero-new-dependency constraint is verified
explicitly in Task 8, Step 5.

**Placeholder scan.** No TBDs. Every code step carries compilable code. Every lavaplayer
signature used in the plan was read off `lavaplayer-2.2.7.jar` with `javap`, so no step
depends on a guessed API.

**Type consistency.** `BilibiliVideo`, `BilibiliAudioStream`, and `BilibiliLink` keep the
same accessors across all tasks. `cid` is `long` throughout. `durationMs` is milliseconds
everywhere, matching lavaplayer's `AudioTrackInfo.length`. The search path produces
`cid == 0`, and Task 5 states the guard `loadAudioStream` needs for it.
