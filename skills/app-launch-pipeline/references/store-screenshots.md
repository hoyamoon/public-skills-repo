# 스토어 스크린샷·그래픽 자산 생성 (STEP 5 필수 절차)

Play Console 등록정보는 **폰 + 7인치 태블릿 + 10인치 태블릿 스크린샷이 모두 필수(\*)** 다.
폰 것만 만들면 등록정보 저장이 막힌다(실측). 아래 절차로 3종을 한 번에 만든다.

## 0. 규격 (콘솔 문구로 매번 재확인할 것)

| 자산 | 규격 | 비고 |
|---|---|---|
| 앱 아이콘 | 512×512 PNG/JPEG, ≤1MB | 필수 |
| 그래픽 이미지(피처) | 1024×500 PNG/JPEG, ≤15MB | 필수 |
| 휴대전화 스크린샷 | 2~8장, 장변 ≤ 단변×2, 320~3840px | 필수 |
| 7인치 태블릿 | 4~8장, 동일 비율 규칙 | **필수** |
| 10인치 태블릿 | 4~8장, 1080~7680px | **필수** |

**핵심 함정**: 요즘 폰 원본(1080×2400 = 2.22:1)은 **비율 규격 위반**이다.
1080×1920(16:9) 캔버스에 캡션 바 + 축소 배치로 재합성하면 규격을 맞추면서 전환율도 올린다.

## 1. 폰 스크린샷 — 계측 테스트로 전 화면 캡처

액티비티 대부분은 `exported=false`라 `adb am start`로 직접 못 띄운다.
`ActivityScenario`를 쓰는 계측 테스트로 캡처한다.

```kotlin
// app/src/androidTest/.../ScreenshotFlowTest.kt
private fun shot(name: String) {
    Thread.sleep(1200)
    val bmp = InstrumentationRegistry.getInstrumentation().uiAutomation.takeScreenshot() ?: return
    val dir = File(ctx.getExternalFilesDir(null), "shots").apply { mkdirs() }
    File(dir, "$name.png").outputStream().use { bmp.compress(Bitmap.CompressFormat.PNG, 90, it) }
}
@Test fun t1_main() { ActivityScenario.launch(MainActivity::class.java).use { shot("01_home") } }
```

앱 언어를 강제해 타깃 로케일 UI로 찍는다(Android 13+, 시스템 로케일은 안 바뀐다):
```bash
adb shell cmd locale set-app-locales <pkg> --locales ko-KR
```

**빌드 산출물 파일명을 확인한다.** `archivesBaseName`을 설정한 프로젝트는 `app-debug.apk`가
아니라 `<name>-debug.apk`로 나온다 — `find app/build/outputs/apk -name "*.apk"`로 확인하고 설치한다.

**잠금화면 컷이 필요하면 잠금화면을 먼저 켠다.** 꺼져 있으면 전원 버튼을 눌러도 홈 화면이 찍힌다
(실측 — 첫 시도에 홈 화면이 찍혔다).
```bash
adb shell locksettings set-disabled false   # 스와이프 잠금 활성
adb shell input keyevent 26; sleep 2        # 재우기
adb shell input keyevent 26; sleep 3        # 깨우기 → 잠금화면
adb exec-out screencap -p > lock.png
adb shell input keyevent 82                 # 해제
adb shell locksettings set-disabled true    # 원복
```

### 1.1 다국어 리스팅이면 **데모 데이터까지** 그 언어로

UI만 번역되고 화면 안의 값이 원래 언어로 남으면 해외 리스팅에서 곧바로 어색해진다.
시드 데이터를 하드코딩하지 말고 **계측 인자로 로케일 분기**한다.

```kotlin
// androidTest/.../DemoData.kt
object DemoData {
    private fun lang(ctx: Context): String =
        InstrumentationRegistry.getArguments().getString("lang")
            ?: ctx.resources.configuration.locales[0].language
    fun of(ctx: Context) = forLang(lang(ctx))
    fun forLang(lang: String) = when (lang) { "en" -> …; "ja" -> …; else -> … }
}
```
```bash
adb shell am instrument -w -e lang ja -e class <pkg>.ScreenshotFlowTest …
```
말장난처럼 직역이 안 되는 값은 언어별 대체안을 사용자에게 확정받는다.

### 1.2 1번 컷은 가능하면 실기기로

에뮬레이터 화면과 실기기 실사용 화면은 설득력 차이가 크다. `concept_promise`를 한 장으로
보여주는 1번 컷만이라도 실기기로 찍는 게 낫다.

문제는 언어별로 찍으려면 화면의 데이터를 매번 다시 입력해야 한다는 것이다.
**앱에 가져오기(import) 기능이 있으면 언어별 시드 데이터를 파일이나 코드로 건네 한 번에
채우고, 촬영 후 원래대로 되돌린다.**

주의 하나 — **앱이 이미지를 생성해 저장하는 기능이 있다면, 그 이미지는 생성 시점의 언어로
구워진다.** 앱 언어만 바꾸고 기존 이미지를 그대로 두면 이전 언어가 남는다.
언어 변경 → 이미지 재생성 → 촬영 순서를 지킨다.

## 2. 태블릿 스크린샷 — 에뮬레이터 해상도를 바꿔 실제 레이아웃 캡처

별도 태블릿 AVD를 만들 필요 없다. 기존 폰 AVD의 해상도·밀도만 바꾸면
**진짜 태블릿 레이아웃이 렌더링**된다(합성이 아니라 실제 캡처라 정직하다).

```bash
run_shots() {  # $1=size $2=density $3=출력폴더
  adb shell wm size $1; adb shell wm density $2; sleep 5
  adb shell rm -f /sdcard/Android/data/<pkg>/files/shots/*.png   # 이전 캡처만 정리
  adb shell am instrument -w -e class <pkg>.ScreenshotFlowTest <pkg>.test/androidx.test.runner.AndroidJUnitRunner
  adb pull /sdcard/Android/data/<pkg>/files/shots/ $3/
}
run_shots "1200x1920" "213" t7     # 7인치
run_shots "1600x2560" "240" t10    # 10인치
adb shell wm size reset; adb shell wm density reset   # 반드시 원복
```

**함정 2가지**
- 화면 잠금이 켜져 있으면 잠금화면이 찍힌다 → `adb shell locksettings set-disabled true` 후 캡처.
  (단 잠금화면 컷 자체가 필요한 앱이면 이걸 역이용해 1번 컷으로 쓴다)
- 해상도 변경 후 앱 프로세스가 살아 있으면 옛 레이아웃이 남는다 → 캡처 전 `sleep 5`.

## 3. Play 규격 합성 (캡션 바 + 라운드 프레임)

준비: `pip install pillow`. 폰트 경로는 OS마다 다르므로 §3.1의 표에서 골라 쓴다.

```python
from PIL import Image, ImageDraw, ImageFont
BG = (22, 24, 29)   # 앱 브랜드 다크 뉴트럴
FONT_KO = "..."     # §3.1 표에서 자기 OS 경로로 채운다
def build(src_path, caption, out_path, W, H):
    src = Image.open(src_path).convert("RGB")
    canvas = Image.new("RGB", (W, H), BG); d = ImageDraw.Draw(canvas)
    f = ImageFont.truetype(FONT_KO, int(W*0.045))
    y = int(H*0.03)
    for line in caption.split("\n"):
        bb = d.textbbox((0,0), line, font=f)
        d.text(((W-(bb[2]-bb[0]))//2, y), line, font=f, fill=(255,255,255)); y += int(W*0.06)
    cap_h = int(H*0.16); avail = H - cap_h - int(H*0.03)
    scale = min((W-W*0.13)/src.width, avail/src.height)
    nw, nh = int(src.width*scale), int(src.height*scale)
    thumb = src.resize((nw, nh), Image.LANCZOS)
    mask = Image.new("L", (nw, nh), 0)
    ImageDraw.Draw(mask).rounded_rectangle([0,0,nw,nh], radius=int(W*0.035), fill=255)
    canvas.paste(thumb, ((W-nw)//2, cap_h), mask); canvas.save(out_path, "PNG")
```

캔버스 크기: 폰 `1080×1920`, 7인치 `1200×1920`, 10인치 `1600×2560`.
한국어 캡션은 의미 단위("두 진행자", "1080p 영상")가 줄 경계에서 찢어지지 않게 직접 개행한다.

**규격 자체검증을 반드시 넣는다**:
```python
assert max(im.size) <= min(im.size)*2 and min(im.size) >= 320
```

### 3.1 🔴 캡션 폰트 — 언어마다 글리프가 있는지 실제로 렌더해서 본다

**한 폰트가 모든 언어를 커버하지 않는다.** 한국어 폰트(macOS AppleSDGothicNeo)에는 일본어
신자체 글리프가 없어 `写`·`変` 같은 글자가 두부(□)로 나온다(실측).

OS별 기본 경로다. **자기 환경에 실제로 그 파일이 있는지 먼저 확인하고 쓴다.**

| OS | 한국어 | 일본어 |
|---|---|---|
| macOS | `/System/Library/Fonts/AppleSDGothicNeo.ttc` | `/System/Library/Fonts/Hiragino Sans GB.ttc` |
| Windows | `C:\Windows\Fonts\malgun.ttf` | `C:\Windows\Fonts\meiryo.ttc` |
| Linux | `/usr/share/fonts/.../NotoSansKR-Regular.otf` | `/usr/share/fonts/.../NotoSansJP-Regular.otf` |

OS를 안 타는 방법이 하나 있다. **Noto Sans CJK를 프로젝트에 직접 받아 두는 것**이다
(구글 Noto 프로젝트, OFL 라이선스). 한·중·일을 한 파일이 커버해서 언어 분기가 사라지고,
다른 컴퓨터에서 다시 만들 때도 경로가 안 깨진다. 배포용 스크립트라면 이쪽을 권한다.

더 중요한 건 **검사 방법**이다. PIL의 `font.getmask(ch).getbbox() is None` 검사는
글리프가 없어도 통과한다 — 이 검사만 믿으면 깨진 이미지를 그대로 올리게 된다.
**언어별 샘플 문자열을 실제로 렌더해 이미지로 확인**하는 단계를 반드시 넣는다.

```python
FONT_BY_LANG = {"ko": FONT_KO, "ja": FONT_JA}   # 위 표에서 자기 OS 경로로 채운다
# 각 언어 대표 문자열을 한 장에 렌더해 눈으로 확인한 뒤 본작업에 들어간다
# 예: ko "무음 촬영 · 1080p 영상" / ja "写真を無音で撮影 · 変換"
```

## 4. 아이콘 — 앱 리소스에서 렌더한다 (손으로 그리지 않는다)

**스토어 512 아이콘을 따로 그리면 앱 아이콘과 어긋난다.** 런처 아이콘을 바꾼 뒤에도
예전 512가 남아 있는 사고가 실제로 났다. 앱에 들어간 리소스에서 렌더하면 구조적으로
어긋날 수 없다.

```kotlin
val d = ctx.packageManager.getApplicationIcon(ctx.packageName)
val bmp = Bitmap.createBitmap(512, 512, Bitmap.Config.ARGB_8888)
val c = Canvas(bmp)
// 어댑티브 아이콘을 그냥 draw() 하면 런처 마스크(에뮬레이터는 원형)가 먹어 모서리가 검게 남는다.
// Play 는 자체적으로 모서리를 처리하므로 배경·전경 레이어를 마스크 없이 정사각 전체에 그린다.
if (d is AdaptiveIconDrawable) {
    d.background?.apply { setBounds(0, 0, 512, 512); draw(c) }
    d.foreground?.apply { setBounds(0, 0, 512, 512); draw(c) }
} else { d.setBounds(0, 0, 512, 512); d.draw(c) }
// 검증: 모서리 픽셀 alpha == 255 여야 한다 (마스크가 먹었으면 투명/검정)
```

아이콘은 **48px로 축소한 확인본을 함께 만들어** 런처·검색결과 가독성을 눈으로 검증한다.

### 4.1 홍보 이미지에 들어간 상징·심볼을 점검한다

생성 이미지에 **인증 표장·기관 로고가 섞여 들어올 수 있다.** 실제로 구조 인력이 등장하는
피처 그래픽에서 제복 어깨에 Star of Life(6각 별 + 아스클레피오스 지팡이 — 미국 인증 표장)가
그려져 나온 적이 있다. 적십자, 소방·경찰 기관 로고, 병원 브랜드도 같은 범주다.

- 최종 이미지를 **확대해서 제복·배지·간판 영역을 확인**한다
- 지워야 하면 마스크를 **원판 전체가 아니라 '해당 색 픽셀'로 한정**한다. 원판째 지우면
  옆의 팔·옷 경계까지 뭉개진다. 주변 색을 확산(라플라스 인페인팅)시키면 음영이 이어진다
- 피부처럼 비슷한 색이 섞여 있으면 `|g-b|`로 구분한다(패치 분홍은 g≈b, 피부는 b가 낮다)
- 검증: 대상 영역 잔여 픽셀 0 / **마스크 밖 픽셀 변화 0** / 인접 요소 픽셀 수 유지

## 5. 업로드 — 이미지는 사람이 올린다

브라우저 자동화의 파일 업로드는 세션에 공유된 파일만 허용하는 경우가 많아 프로젝트
산출물을 그대로 못 올린다. → 이미지는 파일로 만들어 두고 폴더를 열어 사용자가 드래그하게 한다.
(AAB는 예외 — Gradle Play Publisher 등으로 API 업로드가 된다)

**연출 이미지를 스크린샷 슬롯에 넣지 않는다.** Play는 스크린샷을 실제 인앱 경험 위주로
채우고 과도한 연출을 피하라고 안내한다. 홍보용 일러스트를 굳이 쓴다면 실제 UI 컷을 앞에
두고 맨 뒤에 배치한다 — 지적받으면 그 컷만 내리면 되게.

## 6. 업로드 후 — 실제로 뭐가 올라갔는지 API로 확인한다

사람이 수동 업로드하면 "무엇이 반영됐는지" 아무도 모르는 상태가 된다. Play Developer API로
**읽기 전용 확인**을 한다(edit을 만들되 commit하지 않고 delete하면 아무것도 바뀌지 않는다).

```
POST   /applications/{pkg}/edits               # edit 생성
GET    /edits/{id}/tracks                      # 트랙별 versionCode·status
GET    /edits/{id}/bundles                     # 업로드된 AAB 목록(sha1 포함)
GET    /edits/{id}/listings                    # 언어별 제목·설명
GET    /edits/{id}/listings/{lang}/{imageType} # icon·featureGraphic·phoneScreenshots 등 장수
DELETE /edits/{id}                             # 반드시 폐기 — 읽기 전용 보장
```

- **업로드된 AAB가 최신인지**: `bundles`의 `sha1`과 로컬 `shasum -a 1 *.aab` 비교
- **언어별 이미지 누락**: `icon`·`featureGraphic`이 0개인 언어는 기본 언어로 폴백된다(정상)
- **장수가 예상과 다르면** 옛 컷이 남아 있는 것 — 실제 컷 수와 대조한다

> ⚠️ API로 할 수 있는 것은 여기까지다. **검토 제출과 게시는 API에 없다** — 콘솔에서 사람이
> 눌러야 한다. 앱 콘텐츠 선언 상태도 API에 노출되지 않는다(SKILL.md §8.4 참조).

## 체크리스트

- [ ] 폰 2~8장 (1080×1920 재합성, 원본 2.22:1 그대로 올리지 않기)
- [ ] 7인치 4~8장 (1200×1920)
- [ ] 10인치 4~8장 (1600×2560)
- [ ] 아이콘 512 = **앱 리소스에서 렌더** + 48px 확인본 + 모서리 불투명 검증
- [ ] 피처 그래픽 1024×500 / 인증 표장·기관 로고 혼입 점검
- [ ] 전 파일 비율·최소변 자체검증 통과
- [ ] 타깃 로케일 UI로 캡처됐는지 육안 확인 + **데모 데이터도 그 언어인지**
- [ ] 언어별 캡션을 실제로 렌더해 두부(□) 없는지 확인
- [ ] 1번 컷 = `concept_promise`를 한 장으로 보여주는 화면 (가능하면 실기기)
- [ ] 업로드 후 API로 트랙·번들 sha1·언어별 이미지 장수 확인
