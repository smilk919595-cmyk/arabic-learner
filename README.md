# arabic-learner
name: arabic_learner
description: 面向阿语专业学生的阿拉伯语学习助手
publish_to: 'none'
version: 1.0.0+1
environment:
  sdk: '>=3.4.0 <4.0.0'
dependencies:
  flutter:
    sdk: flutter
flutter:
  uses-material-design: true
  name: Build APK
on:
  push:
    branches: [ main ]
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'zulu'
          java-version: '17'
          
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.5'
          channel: 'stable'
          cache: true
      - name: Get dependencies
        run: flutter pub get
      - name: Build APK
        run: flutter build apk --release

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-release-apk
          path: build/app/outputs/flutter-apk/app-release.apk
      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: v1.0.${{ github.run_number }}
          name: 阿拉伯语学习助手 v1.0.${{ github.run_number }}
          files: build/app/outputs/flutter-apk/app-release.apk
          import 'package:flutter/material.dart';

// =====================================================================
// 一、阿拉伯语工具
// =====================================================================
const String kFatha = '\u064E';
const String kDamma = '\u064F';
const String kKasra = '\u0650';
const String kSukun = '\u0652';
const String kShadda = '\u0651';
const String kFathatan = '\u064B';
const String kDammatan = '\u064C';
const String kKasratan = '\u064D';
const String kAllDiacritics =
    '$kFatha$kDamma$kKasra$kSukun$kShadda$kFathatan$kDammatan$kKasratan';

bool _isDiacritic(String c) => kAllDiacritics.contains(c);

String stripDiacritics(String s) {
  final b = StringBuffer();
  for (final r in s.runes) {
    final c = String.fromCharCode(r);
    if (!_isDiacritic(c)) b.write(c);
  }
  return b.toString();
}

String normalizeArabic(String s) {
  var t = stripDiacritics(s);
  t = t.replaceAll('\u0640', '');
  t = t.replaceAll(RegExp('[\u0622\u0623\u0625]'), '\u0627');
  t = t.replaceAll('\u0649', '\u064A');
  t = t.replaceAll('\u0629', '\u0647');
  return t.trim();
}

String stripFinalShortVowel(String s) {
  if (s.isEmpty) return s;
  final last = s[s.length - 1];
  if (last == kFatha || last == kDamma || last == kKasra ||
      last == kFathatan || last == kDammatan || last == kKasratan) {
    return s.substring(0, s.length - 1);
  }
  return s;
}

String guessCaseByEnding(String v) {
  if (v.isEmpty) return '无显性尾符';
  if (v.endsWith(kDamma)) return '主格（مَرْفُوع），标志：ـُ';
  if (v.endsWith(kDammatan)) return '主格（مَرْفُوع），标志：ـٌ';
  if (v.endsWith(kFatha)) return '宾格（مَنْصُوب），标志：ـَ';
  if (v.endsWith(kFathatan)) return '宾格（مَنْصُوب），标志：ـً';
  if (v.endsWith(kKasra)) return '属格（مَجْرُور），标志：ـِ';
  if (v.endsWith(kKasratan)) return '属格（مَجْرُور），标志：ـٍ';
  if (v.endsWith(kSukun)) return '定格（سَاكِن），多见于动词或虚词';
  return '无显性尾符（需人工核对）';
}

// =====================================================================
// 二、数据模型
// =====================================================================
class ExampleSentence {
  final String ar;
  final String zh;
  const ExampleSentence(this.ar, this.zh);
}

class WordEntry {
  final String ar, root, pos, zh, gender, singular, plural, plural2;
  final String past, present, transitivity;
  final List<String> family, phrases;
  final List<ExampleSentence> examples;
  const WordEntry({
    required this.ar,
    this.root = '',
    this.pos = '',
    required this.zh,
    this.gender = '',
    this.singular = '',
    this.plural = '',
    this.plural2 = '',
    this.past = '',
    this.present = '',
    this.transitivity = '',
    this.family = const [],
    this.phrases = const [],
    this.examples = const [],
  });
  bool get isVerb => past.isNotEmpty || present.isNotEmpty;
  bool get isNoun => singular.isNotEmpty || plural.isNotEmpty;
}

class KnowledgeItem {
  final String id, category, title, subtitle;
  final List<String> points;
  final List<ExampleSentence> examples;
  const KnowledgeItem({
    required this.id,
    required this.category,
    required this.title,
    this.subtitle = '',
    this.points = const [],
    this.examples = const [],
  });
}

class ParticleItem {
  final String word, type, usage;
  final List<ExampleSentence> examples;
  const ParticleItem({
    required this.word,
    required this.type,
    required this.usage,
    this.examples = const [],
  });
}

class FavItem {
  final String type, title, content, note;
  const FavItem({
    required this.type,
    required this.title,
    required this.content,
    this.note = '',
  });
  String get key => '$type|$title|$content';
}

class FavoritesStore extends ChangeNotifier {
  FavoritesStore._();
  static final FavoritesStore instance = FavoritesStore._();
  final List<FavItem> _items = [];
  List<FavItem> get items => List.unmodifiable(_items);
  bool contains(FavItem it) => _items.any((e) => e.key == it.key);
  void toggle(FavItem it) {
    final i = _items.indexWhere((e) => e.key == it.key);
    if (i >= 0) {
      _items.removeAt(i);
    } else {
      _items.insert(0, it);
    }
    notifyListeners();
  }
  void removeAt(int i) {
    if (i < 0 || i >= _items.length) return;
    _items.removeAt(i);
    notifyListeners();
  }
  void clear() {
    _items.clear();
    notifyListeners();
  }
}

// =====================================================================
// 三、词典数据
// =====================================================================
const List<WordEntry> kDictionary = [
  WordEntry(
    ar: 'كِتَابٌ', root: 'ك ت ب', pos: '名词', zh: '书；书籍',
    gender: '阳性', singular: 'كِتَابٌ',
    plural: 'كُتُبٌ（破碎复数）', plural2: 'كِتَابَاتٌ（完整复数，少用）',
    family: ['كَتَبَ 写', 'كَاتِبٌ 作家', 'مَكْتُوبٌ 被写的', 'مَكْتَبٌ 办公室', 'مَكْتَبَةٌ 图书馆'],
    phrases: ['كِتَابُ اللُّغَةِ الْعَرَبِيَّةِ 阿拉伯语课本'],
    examples: [
      ExampleSentence('هَذَا كِتَابٌ جَدِيدٌ.', '这是一本新书。'),
      ExampleSentence('أَقْرَأُ الْكِتَابَ فِي الْمَكْتَبَةِ.', '我在图书馆里读这本书。'),
    ],
  ),
  WordEntry(
    ar: 'كَتَبَ', root: 'ك ت ب', pos: '动词', zh: '写；书写',
    past: 'كَتَبَ', present: 'يَكْتُبُ',
    transitivity: '及物动词（مُتَعَدٍّ）',
    family: ['كِتَابٌ 书', 'كَاتِبٌ 作家', 'مَكْتُوبٌ 被写的'],
    phrases: ['كَتَبَ الدَّرْسَ 写课文'],
    examples: [
      ExampleSentence('كَتَبَ الطَّالِبُ الدَّرْسَ.', '学生写了课文。'),
      ExampleSentence('أَكْتُبُ الْوَاجِبَ كُلَّ يَوْمٍ.', '我每天写作业。'),
    ],
  ),
  WordEntry(
    ar: 'طَالِبٌ', root: 'ط ل ب', pos: '名词', zh: '学生',
    gender: '阳性（阴性：طَالِبَةٌ）', singular: 'طَالِبٌ',
    plural: 'طُلَّابٌ（破碎复数）', plural2: 'طَلَبَةٌ',
    family: ['طَلَبَ 要求', 'طَلَبٌ 请求'],
    phrases: ['طَالِبُ الْعِلْمِ 求知者'],
    examples: [
      ExampleSentence('أَنَا طَالِبٌ فِي كُلِّيَّةِ اللُّغَاتِ.', '我是外语学院的学生。'),
    ],
  ),
  WordEntry(
    ar: 'مَدْرَسَةٌ', root: 'د ر س', pos: '名词', zh: '学校',
    gender: '阴性', singular: 'مَدْرَسَةٌ',
    plural: 'مَدَارِسُ（破碎复数）', plural2: 'مَدْرَسَاتٌ（完整复数）',
    family: ['دَرَسَ 学习', 'دِرَاسَةٌ 学习', 'مُدَرِّسٌ 教师'],
    phrases: ['مَدْرَسَةٌ ثَانَوِيَّةٌ 中学'],
    examples: [
      ExampleSentence('أَذْهَبُ إِلَى الْمَدْرَسَةِ صَبَاحًا.', '我早上去学校。'),
    ],
  ),
  WordEntry(
    ar: 'بَيْتٌ', root: 'ب ي ت', pos: '名词', zh: '房子；家',
    gender: '阳性', singular: 'بَيْتٌ',
    plural: 'بُيُوتٌ（破碎复数）', plural2: 'أَبْيَاتٌ',
    family: ['بَاتَ 过夜'],
    phrases: ['بَيْتُ اللَّهِ 天房'],
    examples: [
      ExampleSentence('بَيْتِي قَرِيبٌ مِنَ الْمَدْرَسَةِ.', '我家离学校很近。'),
    ],
  ),
  WordEntry(
    ar: 'قَرَأَ', root: 'ق ر أ', pos: '动词', zh: '读；阅读',
    past: 'قَرَأَ', present: 'يَقْرَأُ', transitivity: '及物动词',
    family: ['قِرَاءَةٌ 阅读', 'قَارِئٌ 读者'],
    phrases: ['قَرَأَ الْكِتَابَ 读书'],
    examples: [
      ExampleSentence('قَرَأْتُ الْقُرْآنَ الْيَوْمَ.', '我今天读了《古兰经》。'),
    ],
  ),
  WordEntry(
    ar: 'ذَهَبَ', root: 'ذ ه ب', pos: '动词', zh: '去；走',
    past: 'ذَهَبَ', present: 'يَذْهَبُ',
    transitivity: '不及物动词（لَازِمٌ），常带 إِلَى',
    family: ['ذَهَابٌ 前往'],
    phrases: ['ذَهَبَ إِلَى الْمَدْرَسَةِ 去学校'],
    examples: [
      ExampleSentence('ذَهَبَ أَحْمَدُ إِلَى السُّوقِ.', '艾哈迈德去了市场。'),
    ],
  ),
  WordEntry(
    ar: 'أُسْتَاذٌ', root: 'أ س ت ذ', pos: '名词', zh: '老师；教授',
    gender: '阳性（阴性：أُسْتَاذَةٌ）', singular: 'أُسْتَاذٌ',
    plural: 'أَسَاتِذَةٌ（破碎复数）',
    family: ['أُسْتَاذَةٌ 女教师'],
    phrases: ['أُسْتَاذُ اللُّغَةِ الْعَرَبِيَّةِ 阿拉伯语老师'],
    examples: [
      ExampleSentence('أُسْتَاذِي رَجُلٌ فَاضِلٌ.', '我的老师是一位品德高尚的人。'),
    ],
  ),
  WordEntry(
    ar: 'لُغَةٌ', root: 'ل غ و', pos: '名词', zh: '语言',
    gender: '阴性', singular: 'لُغَةٌ',
    plural: 'لُغَاتٌ（完整复数）',
    family: ['لُغَوِيٌّ 语言学的'],
    phrases: ['اللُّغَةُ الْعَرَبِيَّةُ 阿拉伯语'],
    examples: [
      ExampleSentence('اللُّغَةُ الْعَرَبِيَّةُ لُغَةٌ جَمِيلَةٌ.', '阿拉伯语是一门美丽的语言。'),
    ],
  ),
  WordEntry(
    ar: 'مَسْجِدٌ', root: 'س ج د', pos: '名词', zh: '清真寺',
    gender: '阳性', singular: 'مَسْجِدٌ',
    plural: 'مَسَاجِدُ（破碎复数）',
    family: ['سَجَدَ 叩头', 'سُجُودٌ 叩拜'],
    phrases: ['الْمَسْجِدُ الْحَرَامُ 禁寺'],
    examples: [
      ExampleSentence('نُصَلِّي فِي الْمَسْجِدِ.', '我们在清真寺里礼拜。'),
    ],
  ),
];

// =====================================================================
// 四、虚词数据
// =====================================================================
const List<ParticleItem> kParticleItems = [
  ParticleItem(
    word: 'إِلَّا أَنَّ', type: '复合虚词',
    usage: '表转折或补充，意为「只不过……」。أَنَّ 属类动词虚词，其后名词为宾格，述语为主格。',
    examples: [
      ExampleSentence('الْكِتَابُ سَهْلٌ إِلَّا أَنَّ تَمَارِينَهُ كَثِيرَةٌ.',
          '这本书很容易，只不过它的练习很多。'),
    ],
  ),
  ParticleItem(
    word: 'لَكِنَّ', type: '类动词虚词',
    usage: '表转折，意为「但是」。其后名词为宾格，述语为主格。',
    examples: [
      ExampleSentence('الطَّالِبُ مُجْتَهِدٌ لَكِنَّهُ مَرِيضٌ.',
          '这个学生很勤奋，但是他生病了。'),
    ],
  ),
  ParticleItem(
    word: 'إِنَّ', type: '类动词虚词',
    usage: '表强调确认，意为「确实」。其后名词为宾格，述语为主格。',
    examples: [
      ExampleSentence('إِنَّ اللَّغَةَ الْعَرَبِيَّةَ وَاسِعَةٌ.', '阿拉伯语确实博大精深。'),
    ],
  ),
  ParticleItem(
    word: 'كَانَ', type: '残缺动词',
    usage: '「曾经是」。只支配主语（主格）和述语（宾格）。姐妹词：أَصْبَحَ / ظَلَّ / أَمْسَى。',
    examples: [
      ExampleSentence('كَانَ الطَّالِبُ مُجْتَهِدًا.', '这个学生（过去）很勤奋。'),
    ],
  ),
  ParticleItem(
    word: 'لَمْ', type: '否定虚词',
    usage: '后接现在式动词，表过去否定，使动词尾符变静符（مَجْزُوم）。',
    examples: [
      ExampleSentence('لَمْ يَذْهَبْ أَحْمَدُ إِلَى الْمَدْرَسَةِ.', '艾哈迈德没有去学校。'),
    ],
  ),
  ParticleItem(
    word: 'لَنْ', type: '否定虚词',
    usage: '后接现在式动词，表将来强调否定，使动词尾符变宾格（مَنْصُوب）。',
    examples: [
      ExampleSentence('لَنْ أَنْسَى فَضْلَكَ.', '我绝不会忘记你的恩情。'),
    ],
  ),
  ParticleItem(
    word: 'لَا', type: '否定虚词',
    usage: '否定现在式动词，或否定名词句（لَا النَّافِيَةُ لِلْجِنْسِ，其名词为宾格）。',
    examples: [
      ExampleSentence('لَا أَفْهَمُ هَذِهِ الْجُمْلَةَ.', '我不理解这个句子。'),
    ],
  ),
  ParticleItem(
    word: 'إِذَا', type: '条件虚词',
    usage: '意为「当……时；如果……就……」，其后多接过去式动词表将来。',
    examples: [
      ExampleSentence('إِذَا دَرَسْتَ نَجَحْتَ.', '如果你学习，你就会成功。'),
    ],
  ),
];

// =====================================================================
// 五、知识库数据
// =====================================================================
const List<KnowledgeItem> kGrammarItems = [
  KnowledgeItem(id: 'g01', category: '名词', title: '名词的性：阳性与阴性',
    subtitle: '《新编阿拉伯语》第一册',
    points: [
      '阿拉伯语名词分阳性（مُذَكَّر）和阴性（مُؤَنَّث）两大类。',
      '没有明显阴性标志的名词一般视为阳性。',
      '常见阴性标志：① 词尾的 ة；② 词尾的 ى；③ 部分自然阴性名词如 أُمٌّ。',
    ],
    examples: [
      ExampleSentence('طَالِبٌ / طَالِبَةٌ', '男学生 / 女学生'),
    ]),
  KnowledgeItem(id: 'g02', category: '名词', title: '名词的数：单数、双数、复数',
    subtitle: '《新编阿拉伯语》第一册',
    points: [
      '阿拉伯语有单数、双数（مُثَنَّى）、复数（جَمْع）三种数。',
      '双数：单数词尾加 انِ（主格）或 يْنِ（宾属格）。',
      '复数分完整复数（加 ونَ/ينَ 或 اتٌ）和破碎复数（内部结构改变，需逐词记忆）。',
    ],
    examples: [
      ExampleSentence('كِتَابٌ → كُتُبٌ', '书（单数）→ 书（破碎复数）'),
    ]),
  KnowledgeItem(id: 'g03', category: '名词', title: '名词的格：主格、宾格、属格',
    subtitle: '《新编阿拉伯语》第一册',
    points: [
      '主格（مَرْفُوع）：ـُ / ـٌ，用于主语、起语、述语。',
      '宾格（مَنْصُوب）：ـَ / ـً，用于宾语、كان 的述语。',
      '属格（مَجْرُور）：ـِ / ـٍ，用于介词受词、正偏组合的次次。',
      '完整阳性复数：主格 ونَ，宾属格均用 ينَ。',
      '双数：主格 انِ，宾属格均用 يْنِ。',
    ],
    examples: [
      ExampleSentence('قَرَأْتُ الْكِتَابَ.', '我读了这本书。（宾格宾语）'),
      ExampleSentence('فِي الْمَكْتَبَةِ.', '在图书馆里。（属格）'),
    ]),
  KnowledgeItem(id: 'g04', category: '名词', title: '正偏组合（الْإِضَافَة）',
    subtitle: '《新编阿拉伯语》第一册',
    points: [
      '格式：正次（مُضَاف）+ 次次（مُضَافٌ إِلَيْهِ）。',
      '正次不带 ال、不带鼻音符；次次一律为属格。',
    ],
    examples: [
      ExampleSentence('كِتَابُ الطَّالِبِ', '学生的书'),
    ]),
  KnowledgeItem(id: 'g05', category: '名词', title: '形容词与被形容词',
    subtitle: '《新编阿拉伯语》第一册',
    points: [
      '形容词必须与被形容词在性、数、格、限定性四方面一致。',
      '被形容词带 ال 时，形容词也必须带 ال。',
    ],
    examples: [
      ExampleSentence('الطَّالِبُ الْمُجْتَهِدُ', '那个勤奋的学生'),
    ]),
  KnowledgeItem(id: 'g06', category: '动词', title: '动词的三种式',
    subtitle: '《新编阿拉伯语》第一册',
    points: [
      '过去式（الْمَاضِي）：已发生的动作。',
      '现在式（الْمُضَارِع）：词首带 أ / ت / ي / ن 前缀。',
      '命令式（الْأَمْر）：只用于第二人称。',
    ],
    examples: [
      ExampleSentence('كَتَبَ / يَكْتُبُ / اُكْتُبْ', '写（过去 / 现在 / 命令）'),
    ]),
  KnowledgeItem(id: 'g07', category: '动词', title: '过去式动词变位',
    subtitle: '《新编阿拉伯语》第一册',
    points: [
      '规则三母动词过去式词干固定，通过后缀体现人称、性、数。',
      '第三人称阳性单数无后缀；阴性单数加 ـَتْ；阳性复数加 ـُوا。',
      '第一人称：أَنَا 用 ـتُ，نَحْنُ 用 ـنَا。',
    ],
    examples: [
      ExampleSentence('كَتَبْتُ / كَتَبْنَا / كَتَبُوا', '我写 / 我们写 / 他们写'),
    ]),
  KnowledgeItem(id: 'g08', category: '动词', title: '现在式动词变位',
    subtitle: '《新编阿拉伯语》第二册',
    points: [
      '现在式 = 前缀 + 词干 + 后缀。',
      '后缀：单数 ـُ，双数 ـَانِ，阳性复数 ـُونَ，阴性单数 ـِينَ，阴性复数 ـْنَ。',
      '三格：主格（正常）、宾格（受 لن / أنْ）、定格（受 لم / لا الناهية）。',
    ],
    examples: [
      ExampleSentence('لَنْ أَكْتُبَ / لَمْ أَكْتُبْ', '我绝不写 / 我没有写'),
    ]),
  KnowledgeItem(id: 'g09', category: '动词', title: '命令式动词构成',
    subtitle: '《新编阿拉伯语》第二册',
    points: [
      '命令式由现在式去掉前缀，并在需要时补连读的 ا 构成。',
      '若词干以「辅音+静符」开头，补 اِ（第二母为合口符时补 اُ）。',
    ],
    examples: [
      ExampleSentence('يَكْتُبُ → اُكْتُبْ', '他写 → 你写！'),
      ExampleSentence('يَجْلِسُ → اِجْلِسْ', '他坐 → 你坐！'),
    ]),
  KnowledgeItem(id: 'g10', category: '动词', title: '及物动词与不及物动词',
    subtitle: '《新编阿拉伯语》第二册',
    points: [
      '及物动词（مُتَعَدٍّ）可直接带宾格宾语，如 كَتَبَ。',
      '不及物动词（لَازِمٌ）需借助介词，如 ذَهَبَ إِلَى。',
      '有些动词带两个宾语，如 أَعْطَى（给）。',
    ],
    examples: [
      ExampleSentence('كَتَبَ الطَّالِبُ الدَّرْسَ.', '学生写了课文。'),
    ]),
  KnowledgeItem(id: 'g11', category: '动词', title: '被动语态',
    subtitle: '《新编阿拉伯语》第三册',
    points: [
      '过去式被动：中间元音变齐齿符，其余变合口符，如 كُتِبَ。',
      '现在式被动：前缀变合口符，词中元音保持原样，如 يُكْتَبُ。',
    ],
    examples: [
      ExampleSentence('كُتِبَتِ الرِّسَالَةُ.', '这封信被写了。'),
    ]),
  KnowledgeItem(id: 'g12', category: '虚词与介词', title: '常见介词及其支配的属格',
    subtitle: '《新编阿拉伯语》第一、二册',
    points: [
      '介词一律支配其后名词为属格。',
      '常见：فِي / مِنْ / إِلَى / عَلَى / عَنْ / مَعَ / بِ / لِ / كَ。',
      '介词短语多作定语或状语。',
    ],
    examples: [
      ExampleSentence('الْكِتَابُ عَلَى الْمَكْتَبِ.', '书在办公桌上。'),
    ]),
  KnowledgeItem(id: 'g13', category: '虚词与介词', title: '类动词虚词 إِنَّ 及其姐妹词',
    subtitle: '《新编阿拉伯语》第二册',
    points: [
      'إِنَّ / أَنَّ / كَأَنَّ / لَكِنَّ / لَعَلَّ / لَيْتَ 均属类动词虚词。',
      '其后名词为宾格，述语为主格。',
    ],
    examples: [
      ExampleSentence('إِنَّ الطَّالِبَ مُجْتَهِدٌ.', '这个学生确实很勤奋。'),
    ]),
  KnowledgeItem(id: 'g14', category: '虚词与介词', title: '残缺动词 كَانَ 及其姐妹词',
    subtitle: '《新编阿拉伯语》第三册',
    points: [
      '只支配主语（主格）和述语（宾格）。',
      '常见：كَانَ / أَصْبَحَ / ظَلَّ / أَمْسَى / بَاتَ / لَيْسَ。',
    ],
    examples: [
      ExampleSentence('أَصْبَحَ الْجَوُّ بَارِدًا.', '天气变冷了。'),
    ]),
  KnowledgeItem(id: 'g15', category: '句法', title: '名词句与动词句',
    subtitle: '《新编阿拉伯语》第一、二册',
    points: [
      '名词句：起语（主格）+ 述语（主格）。',
      '动词句：动词 + 主语（主格）+ 宾语（宾格）。',
    ],
    examples: [
      ExampleSentence('الْجَوُّ جَمِيلٌ.', '天气很好。（名词句）'),
      ExampleSentence('كَتَبَ الطَّالِبُ الدَّرْسَ.', '学生写了课文。（动词句）'),
    ]),
  KnowledgeItem(id: 'g16', category: '句法', title: '关系代词与关系从句',
    subtitle: '《新编阿拉伯语》第三册',
    points: [
      '关系代词须与先行词在性、数上一致。',
      '常见：الَّذِي / الَّتِي / الَّذِينَ / اللَّاتِي。',
      '后接关系从句，从句中需含指代先行词的连接代词。',
    ],
    examples: [
      ExampleSentence('الطَّالِبُ الَّذِي يَدْرُسُ بِجِدٍّ نَاجِحٌ.',
          '那个努力学习的学生是成功的。'),
    ]),
  KnowledgeItem(id: 'g17', category: '句法', title: '条件句（إِنْ / إِذَا）',
    subtitle: '《新编阿拉伯语》第三册',
    points: [
      '由条件工具词 + 条件句 + 应答句构成。',
      'إِنْ 后动词可定格；إِذَا 后接过去式表将来。',
      '应答句常以 فَـ 引出。',
    ],
    examples: [
      ExampleSentence('إِنْ تَدْرُسْ تَنْجَحْ.', '如果你学习，你就会成功。'),
    ]),
  KnowledgeItem(id: 'g18', category: '构词法', title: '三字母词根与派生',
    subtitle: '《新编阿拉伯语》第一、二册',
    points: [
      '绝大多数阿语词由三字母词根派生。',
      '常见模式：فَعَلَ / فَاعِلٌ / مَفْعُولٌ / مَفْعَلٌ / فِعَالَةٌ。',
    ],
    examples: [
      ExampleSentence('كَتَبَ → كَاتِبٌ → مَكْتُوبٌ → مَكْتَبَةٌ',
          '写 → 作家 → 被写的 → 图书馆'),
    ]),
];

const List<KnowledgeItem> kHistoryItems = [
  KnowledgeItem(id: 'h01', category: '阿拉伯历史', title: '阿拉伯半岛与蒙昧时代',
    subtitle: '伊斯兰教兴起之前',
    points: [
      '半岛大部分为沙漠与绿洲，居民以贝都因人和城镇居民为主。',
      '南部有也门文明，北部有帕尔米拉、纳巴泰文明。',
      '麦加因克尔白成为宗教与商业中心。',
      '「蒙昧时代」指伊斯兰教兴起前的多神信仰时期。',
    ]),
  KnowledgeItem(id: 'h02', category: '伊斯兰历史', title: '先知穆罕默德生平',
    subtitle: '约 570 — 632 年',
    points: [
      '约 570 年生于麦加古莱什部落哈希姆家族。',
      '610 年在希拉山洞首次接受启示。',
      '622 年迁徙麦地那（希吉拉），为伊斯兰历元年。',
      '630 年光复麦加。',
      '632 年在麦地那归真。',
    ]),
  KnowledgeItem(id: 'h03', category: '伊斯兰历史', title: '四大哈里发时期',
    subtitle: '632 — 661 年',
    points: [
      '艾布·伯克尔（632-634）：平定叛乱，开始编纂《古兰经》。',
      '欧麦尔（634-644）：大规模对外征服，建立伊斯兰历。',
      '奥斯曼（644-656）：定本《古兰经》。',
      '阿里（656-661）：迁都库法。',
    ]),
  KnowledgeItem(id: 'h04', category: '阿拉伯历史', title: '倭马亚王朝',
    subtitle: '661 — 750 年，定都大马士革',
    points: [
      '穆阿维叶建立，开启世袭王朝。',
      '疆域东至中亚，西至西班牙。',
      '阿拉伯语被定为官方语言。',
      '750 年被阿拔斯家族推翻。',
    ]),
  KnowledgeItem(id: 'h05', category: '阿拉伯历史', title: '阿拔斯王朝',
    subtitle: '750 — 1258 年，定都巴格达',
    points: [
      '762 年曼苏尔定都巴格达。',
      '哈伦·拉希德与马蒙时期为鼎盛期（伊斯兰黄金时代）。',
      '830 年设立智慧宫，组织大规模翻译运动。',
      '1258 年蒙古军队攻陷巴格达，王朝灭亡。',
    ]),
  KnowledgeItem(id: 'h06', category: '阿拉伯语', title: '阿拉伯语的历史地位',
    subtitle: '语言史视角',
    points: [
      '属闪米特语系，与希伯来语同源。',
      '《古兰经》语言奠定标准阿拉伯语基础。',
      '28 个字母，从右向左书写。',
      '联合国六种官方工作语言之一。',
    ]),
];

// =====================================================================
// 六、动词变位生成器
// =====================================================================
class ConjugationRow {
  final String pronoun, form;
  const ConjugationRow(this.pronoun, this.form);
}

class ConjugationTable {
  final String pastTitle, presentTitle, imperativeTitle;
  final List<ConjugationRow> pastRows, presentRows, imperativeRows;
  const ConjugationTable({
    required this.pastTitle,
    required this.presentTitle,
    required this.imperativeTitle,
    required this.pastRows,
    required this.presentRows,
    required this.imperativeRows,
  });
}

ConjugationTable buildConjugation(String pastInput, String presentInput) {
  final past = pastInput.trim();
  final present = presentInput.trim();
  final pastStem = stripFinalShortVowel(past);

  var pStem = present;
  if (pStem.length >= 2) pStem = pStem.substring(2);
  pStem = stripFinalShortVowel(pStem);

  final pastRows = <ConjugationRow>[
    ConjugationRow('هُوَ', '$pastStem$kFatha'),
    ConjugationRow('هُمَا', '${pastStem}َا'),
    ConjugationRow('هُمْ', '${pastStem}ُوا'),
    ConjugationRow('هِيَ', '${pastStem}َتْ'),
    ConjugationRow('هُمَا（阴性）', '${pastStem}َتَا'),
    ConjugationRow('هُنَّ', '${pastStem}ْنَ'),
    ConjugationRow('أَنْتَ', '${pastStem}ْتَ'),
    ConjugationRow('أَنْتُمَا', '${pastStem}ْتُمَا'),
    ConjugationRow('أَنْتُمْ', '${pastStem}ْتُمْ'),
    ConjugationRow('أَنْتِ', '${pastStem}ْتِ'),
    ConjugationRow('أَنْتُنَّ', '${pastStem}ْتُنَّ'),
    ConjugationRow('أَنَا', '${pastStem}ْتُ'),
    ConjugationRow('نَحْنُ', '${pastStem}ْنَا'),
  ];

  final presentRows = <ConjugationRow>[
    ConjugationRow('هُوَ', 'يَ${pStem}ُ'),
    ConjugationRow('هُمَا', 'يَ${pStem}َانِ'),
    ConjugationRow('هُمْ', 'يَ${pStem}ُونَ'),
    ConjugationRow('هِيَ', 'تَ${pStem}ُ'),
    ConjugationRow('هُمَا（阴性）', 'تَ${pStem}َانِ'),
    ConjugationRow('هُنَّ', 'يَ${pStem}ْنَ'),
    ConjugationRow('أَنْتَ', 'تَ${pStem}ُ'),
    ConjugationRow('أَنْتُمَا', 'تَ${pStem}َانِ'),
    ConjugationRow('أَنْتُمْ', 'تَ${pStem}ُونَ'),
    ConjugationRow('أَنْتِ', 'تَ${pStem}ِينَ'),
    ConjugationRow('أَنْتُنَّ', 'تَ${pStem}ْنَ'),
    ConjugationRow('أَنَا', 'أَ${pStem}ُ'),
    ConjugationRow('نَحْنُ', 'نَ${pStem}ُ'),
  ];

  var impBase = pStem;
  var hamza = 'اِ';
  if (impBase.length >= 2 && impBase[1] == kSukun) {
    impBase = impBase[0] + impBase.substring(2);
    final v = impBase.length > 2 ? impBase[2] : '';
    hamza = (v == kDamma) ? 'اُ' : 'اِ';
  }
  final impStem = '$hamza$impBase';

  final imperativeRows = <ConjugationRow>[
    ConjugationRow('أَنْتَ', '${impStem}ْ'),
    ConjugationRow('أَنْتُمَا', '${impStem}َا'),
    ConjugationRow('أَنْتُمْ', '${impStem}ُوا'),
    ConjugationRow('أَنْتِ', '${impStem}ِي'),
    ConjugationRow('أَنْتُنَّ', '${impStem}ْنَ'),
  ];

  return ConjugationTable(
    pastTitle: 'الْمَاضِي（过去式）',
    presentTitle: 'الْمُضَارِع（现在式）',
    imperativeTitle: 'الْأَمْر（命令式）',
    pastRows: pastRows,
    presentRows: presentRows,
    imperativeRows: imperativeRows,
  );
}

// =====================================================================
// 七、句子分析器
// =====================================================================
class WordAnalysis {
  final String surface, pos, caseInfo, role, note;
  const WordAnalysis({
    required this.surface,
    required this.pos,
    required this.caseInfo,
    required this.role,
    this.note = '',
  });
}

class SentenceAnalysis {
  final String voweled, translation;
  final List<WordAnalysis> words;
  final List<String> syntax;
  final bool fromCorpus;
  const SentenceAnalysis({
    required this.voweled,
    required this.translation,
    required this.words,
    required this.syntax,
    this.fromCorpus = false,
  });
}

const Map<String, String> kParticles = {
  'و': '连词：和、与',
  'ف': '连词：于是、然后',
  'ثم': '连词：然后（表先后）',
  'او': '连词：或者',
  'لكن': '连词：但是',
  'الا': '工具词：除了（后名词通常为宾格）',
  'ان': '类动词虚词：确实（后名词为宾格）／条件虚词：如果',
  'كان': '残缺动词：曾经是（主语主格、述语宾格）',
  'في': '介词：在……里（支配属格）',
  'من': '介词：从……（支配属格）',
  'الي': '介词：到……（支配属格）',
  'علي': '介词：在……上（支配属格）',
  'عن': '介词：关于（支配属格）',
  'مع': '介词：和……一起（支配属格）',
  'هل': '疑问虚词：……吗',
  'لا': '否定虚词：不',
  'ما': '否定虚词：不；关系代词：……的东西',
  'لم': '否定虚词：没有（后接现在式，尾符变静符）',
  'لن': '否定虚词：绝不（后接现在式，尾符变宾格）',
  'قد': '强调虚词：确实 / 已经',
  'هذا': '指示代词：这个（阳性单数）',
  'هذه': '指示代词：这个（阴性单数）',
  'ذلك': '指示代词：那个（阳性单数）',
  'الذي': '关系代词：……的（阳性单数）',
  'التي': '关系代词：……的（阴性单数）',
  'اذا': '条件虚词：当……时',
  'ب': '介词：凭借、以（支配属格）',
  'ل': '介词：属于、为了（支配属格）',
  'ك': '介词：像……一样（支配属格）',
};

const Set<String> kPrepositionKeys = {
  'في', 'من', 'الي', 'علي', 'عن', 'مع', 'ب', 'ل', 'ك',
};

const Map<String, String> kPronounTable = {
  'هو': '人称代词：他（阳性单数，主格）',
  'هي': '人称代词：她（阴性单数，主格）',
  'هما': '人称代词：他俩/她俩（双数）',
  'هم': '人称代词：他们（阳性复数）',
  'هن': '人称代词：她们（阴性复数）',
  'انت': '人称代词：你（阳性单数）',
  'انتم': '人称代词：你们（阳性复数）',
  'انا': '人称代词：我（单数）',
  'نحن': '人称代词：我们（复数）',
};

String _cleanKey(String s) =>
    normalizeArabic(s).replaceAll(RegExp(r'[^\u0600-\u06FF]'), '');

WordEntry? lookupEntry(String cleaned) {
  bool eq(String a, String b) => normalizeArabic(a) == b;
  for (final e in kDictionary) {
    if (eq(e.ar, cleaned)) return e;
  }
  if (cleaned.startsWith('ال') && cleaned.length > 3) {
    final s = cleaned.substring(2);
    for (final e in kDictionary) {
      if (eq(e.ar, s)) return e;
    }
  }
  for (final p in ['و', 'ف', 'ب', 'ل', 'ك']) {
    if (cleaned.startsWith(p) && cleaned.length > 2) {
      var s = cleaned.substring(1);
      if (s.startsWith('ال') && s.length > 3) s = s.substring(2);
      for (final e in kDictionary) {
        if (eq(e.ar, s)) return e;
      }
    }
  }
  return null;
}

ExampleSentence? _matchCorpus(String input) {
  final k = _cleanKey(input);
  if (k.isEmpty) return null;
  for (final e in kDictionary) {
    for (final ex in e.examples) {
      if (_cleanKey(ex.ar) == k) return ex;
    }
  }
  for (final p in kParticleItems) {
    for (final ex in p.examples) {
      if (_cleanKey(ex.ar) == k) return ex;
    }
  }
  return null;
}

SentenceAnalysis analyzeSentence(String raw) {
  final text = raw.trim();
  final tokens =
      text.split(RegExp(r'\s+')).where((e) => e.isNotEmpty).toList();
  final words = <WordAnalysis>[];
  final syntax = <String>[];

  bool prevPrep = false;
  bool prevInna = false;
  String? foundVerb;

  for (final t in tokens) {
    final key = _cleanKey(t);

    final particle = kParticles[key];
    if (particle != null) {
      words.add(WordAnalysis(
        surface: t,
        pos: '虚词 / 工具词',
        caseInfo: '虚词无格位变化（مَبْنِيٌّ）',
        role: particle,
      ));
      prevPrep = kPrepositionKeys.contains(key);
      prevInna = (key == 'ان');
      continue;
    }

    final pron = kPronounTable[key];
    if (pron != null) {
      words.add(WordAnalysis(
        surface: t,
        pos: '人称代词',
        caseInfo: prevPrep ? '属格（مَجْرُور）' : '主格（مَرْفُوع）',
        role: prevPrep ? '介词的受词' : '主语 / 起语',
      ));
      prevPrep = false;
      prevInna = false;
      continue;
    }

    final entry = lookupEntry(key);
    if (entry != null) {
      String caseInfo;
      if (prevPrep) {
        caseInfo = '属格（مَجْرُور），因前接介词';
      } else if (prevInna) {
        caseInfo = '宾格（مَنْصُوب），因前接 إِنَّ';
      } else {
        caseInfo = guessCaseByEnding(t);
      }
      String role;
      if (entry.isVerb) {
        role = '谓语 / 述语（فعل）';
        foundVerb ??= entry.ar;
      } else if (prevPrep) {
        role = '介词的受词（مجرور）';
      } else if (prevInna) {
        role = 'إِنَّ 的名词（اسم إنَّ）';
      } else {
        role = '名词性成分（主语 / 起语 / 宾语）';
      }
      words.add(WordAnalysis(
        surface: t,
        pos: entry.pos,
        caseInfo: caseInfo,
        role: role,
        note: entry.zh,
      ));
      prevPrep = false;
      prevInna = false;
      continue;
    }

    words.add(WordAnalysis(
      surface: t,
      pos: '未收录词条',
      caseInfo: guessCaseByEnding(t),
      role: '待人工判定',
      note: '本地词库未收录，建议核对教材',
    ));
    prevPrep = false;
    prevInna = false;
  }

  syntax.add('本句共 ${tokens.length} 个词。');
  if (foundVerb != null) {
    syntax.add('句中出现动词「$foundVerb」，属动词句（جُمْلَةٌ فِعْلِيَّةٌ）：动词 + 主语 + 宾语。');
  } else {
    syntax.add('未识别出动词，判断为名词句（جُمْلَةٌ اسْمِيَّةٌ）：起语 + 述语。');
  }
  if (tokens.any((t) => kPrepositionKeys.contains(_cleanKey(t)))) {
    syntax.add('含介词短语（جَارٌ وَمَجْرُورٌ），介词支配其后名词为属格。');
  }
  if (tokens.any((t) => ['الذي', 'التي'].contains(_cleanKey(t)))) {
    syntax.add('含关系代词，后接关系从句（جُمْلَةٌ مَوْصُولَةٌ）。');
  }
  syntax.add('※ 以上为教材体系下的框架性判断；精确 إعراب 请以教师或权威语法书为准。');

  final corpus = _matchCorpus(text);
  String translation;
  String voweled;
  bool fromCorpus = false;
  if (corpus != null) {
    translation = corpus.zh;
    voweled = corpus.ar;
    fromCorpus = true;
  } else {
    voweled = text;
    final gloss = words
        .where((w) => w.note.isNotEmpty)
        .map((w) => '${stripDiacritics(w.surface)}＝${w.note}')
        .join('；');
    translation = gloss.isEmpty
        ? '【本地规则引擎】未命中标准例句库，暂无整句译文。'
        : '【逐词直译】$gloss';
  }

  return SentenceAnalysis(
    voweled: voweled,
    translation: translation,
    words: words,
    syntax: syntax,
    fromCorpus: fromCorpus,
  );
}

// =====================================================================
// 八、通用组件
// =====================================================================
class ArabicText extends StatelessWidget {
  final String text;
  final double size;
  final Color? color;
  final FontWeight weight;
  final TextAlign align;
  const ArabicText(
    this.text, {
    super.key,
    this.size = 20,
    this.color,
    this.weight = FontWeight.w600,
    this.align = TextAlign.right,
  });
  @override
  Widget build(BuildContext context) {
    return Directionality(
      textDirection: TextDirection.rtl,
      child: Text(
        text,
        textAlign: align,
        style: TextStyle(
          fontSize: size,
          height: 2.0,
          color: color,
          fontWeight: weight,
        ),
      ),
    );
  }
}

class SectionTitle extends StatelessWidget {
  final String text;
  final IconData icon;
  const SectionTitle(this.text, {super.key, this.icon = Icons.label_outline});
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.only(bottom: 8, top: 4),
      child: Row(
        children: [
          Icon(icon, size: 16, color: Theme.of(context).colorScheme.primary),
          const SizedBox(width: 6),
          Text(text,
              style: TextStyle(
                fontSize: 14,
                fontWeight: FontWeight.w700,
                color: Theme.of(context).colorScheme.primary,
              )),
        ],
      ),
    );
  }
}

class InfoRow extends StatelessWidget {
  final String label, value;
  final bool isArabic;
  const InfoRow(this.label, this.value, {super.key, this.isArabic = false});
  @override
  Widget build(BuildContext context) {
    if (value.isEmpty) return const SizedBox.shrink();
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 5),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          SizedBox(
            width: 76,
            child: Text(label,
                style: const TextStyle(fontSize: 13, color: Color(0xFF7A8B87))),
          ),
          Expanded(
            child: isArabic
                ? ArabicText(value, size: 17)
                : Text(value, style: const TextStyle(fontSize: 14, height: 1.6)),
          ),
        ],
      ),
    );
  }
}

class FavButton extends StatelessWidget {
  final FavItem item;
  const FavButton(this.item, {super.key});
  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: FavoritesStore.instance,
      builder: (context, _) {
        final on = FavoritesStore.instance.contains(item);
        return IconButton(
          tooltip: on ? '取消收藏' : '收藏',
          icon: Icon(on ? Icons.star : Icons.star_border,
              color: on ? const Color(0xFFF0A500) : null),
          onPressed: () {
            FavoritesStore.instance.toggle(item);
            ScaffoldMessenger.of(context)
              ..hideCurrentSnackBar()
              ..showSnackBar(SnackBar(
                content: Text(on ? '已取消收藏' : '已加入收藏'),
                duration: const Duration(milliseconds: 900),
                behavior: SnackBarBehavior.floating,
              ));
          },
        );
      },
    );
  }
}

class AppCard extends StatelessWidget {
  final Widget child;
  final EdgeInsets padding;
  const AppCard({
    super.key,
    required this.child,
    this.padding = const EdgeInsets.all(14),
  });
  @override
  Widget build(BuildContext context) {
    return Container(
      width: double.infinity,
      margin: const EdgeInsets.only(bottom: 12),
      padding: padding,
      decoration: BoxDecoration(
        color: Colors.white,
        borderRadius: BorderRadius.circular(14),
        boxShadow: const [
          BoxShadow(color: Color(0x0A000000), blurRadius: 8, offset: Offset(0, 2)),
        ],
      ),
      child: child,
    );
  }
}

// =====================================================================
// 九、页面：词典
// =====================================================================
class DictionaryPage extends StatefulWidget {
  const DictionaryPage({super.key});
  @override
  State<DictionaryPage> createState() => _DictionaryPageState();
}

class _DictionaryPageState extends State<DictionaryPage> {
  final _ctrl = TextEditingController();
  List<WordEntry> _list = kDictionary;

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  void _onChanged(String q) {
    final query = q.trim();
    if (query.isEmpty) {
      setState(() => _list = kDictionary);
      return;
    }
    final nq = normalizeArabic(query);
    setState(() {
      _list = kDictionary.where((e) {
        if (e.zh.contains(query)) return true;
        if (normalizeArabic(e.ar).contains(nq)) return true;
        if (normalizeArabic(e.root).replaceAll(' ', '').contains(
              nq.replaceAll(' ', ''),
            )) return true;
        if (e.family.any((f) => normalizeArabic(f).contains(nq))) return true;
        return false;
      }).toList();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('词典查询')),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.fromLTRB(14, 12, 14, 8),
            child: TextField(
              controller: _ctrl,
              onChanged: _onChanged,
              textInputAction: TextInputAction.search,
              decoration: InputDecoration(
                hintText: '输入阿拉伯语或中文，如 كِتَابٌ / 书',
                prefixIcon: const Icon(Icons.search),
                filled: true,
                fillColor: Colors.white,
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(12),
                  borderSide: BorderSide.none,
                ),
                contentPadding: const EdgeInsets.symmetric(vertical: 4),
              ),
            ),
          ),
          Expanded(
            child: _list.isEmpty
                ? const Center(
                    child: Text('未找到词条\n试试其他写法，或去掉元音符',
                        textAlign: TextAlign.center,
                        style: TextStyle(color: Color(0xFF9AA8A4))),
                  )
                : ListView.builder(
                    padding: const EdgeInsets.fromLTRB(14, 4, 14, 20),
                    itemCount: _list.length,
                    itemBuilder: (context, i) {
                      final e = _list[i];
                      return AppCard(
                        child: InkWell(
                          onTap: () => Navigator.push(
                            context,
                            MaterialPageRoute(
                              builder: (_) => WordDetailPage(entry: e),
                            ),
                          ),
                          child: Row(
                            children: [
                              Expanded(
                                child: Column(
                                  crossAxisAlignment: CrossAxisAlignment.start,
                                  children: [
                                    ArabicText(e.ar,
                                        size: 24,
                                        color: const Color(0xFF1E3A34)),
                                    const SizedBox(height: 4),
                                    Text(
                                      '${e.pos}${e.root.isNotEmpty ? ' · 词根 ${e.root}' : ''}',
                                      style: const TextStyle(
                                          fontSize: 12, color: Color(0xFF8B9A96)),
                                    ),
                                    const SizedBox(height: 2),
                                    Text(e.zh,
                                        maxLines: 2,
                                        overflow: TextOverflow.ellipsis,
                                        style: const TextStyle(fontSize: 14)),
                                  ],
                                ),
                              ),
                              const Icon(Icons.chevron_right,
                                  color: Color(0xFFB9C4C1)),
                            ],
                          ),
                        ),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}

class WordDetailPage extends StatelessWidget {
  final WordEntry entry;
  const WordDetailPage({super.key, required this.entry});
  @override
  Widget build(BuildContext context) {
    final fav = FavItem(
      type: '单词',
      title: entry.ar,
      content: '${entry.zh}\n词根：${entry.root}',
    );
    return Scaffold(
      appBar: AppBar(
        title: Text(entry.zh.split('；').first),
        actions: [FavButton(fav)],
      ),
      body: ListView(
        padding: const EdgeInsets.fromLTRB(14, 12, 14, 30),
        children: [
          AppCard(
            child: Column(
              children: [
                ArabicText(entry.ar,
                    size: 34, align: TextAlign.center,
                    color: const Color(0xFF1E3A34)),
                const SizedBox(height: 8),
                Text(entry.zh,
                    textAlign: TextAlign.center,
                    style: const TextStyle(fontSize: 15)),
                if (entry.root.isNotEmpty) ...[
                  const SizedBox(height: 12),
                  Wrap(
                    alignment: WrapAlignment.center,
                    spacing: 8,
                    children: [
                      Chip(
                        visualDensity: VisualDensity.compact,
                        avatar: const Icon(Icons.account_tree_outlined, size: 16),
                        label: Text('词根 ${entry.root}'),
                      ),
                      Chip(
                        visualDensity: VisualDensity.compact,
                        label: Text(entry.pos),
                      ),
                    ],
                  ),
                ],
              ],
            ),
          ),
          if (entry.isNoun)
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('名词信息', icon: Icons.category_outlined),
                  InfoRow('阴阳性', entry.gender),
                  InfoRow('单数', entry.singular, isArabic: true),
                  InfoRow('破碎复数', entry.plural, isArabic: true),
                  InfoRow('完整复数', entry.plural2, isArabic: true),
                ],
              ),
            ),
          if (entry.isVerb)
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('动词信息', icon: Icons.bolt_outlined),
                  InfoRow('过去式', entry.past, isArabic: true),
                  InfoRow('现在式', entry.present, isArabic: true),
                  InfoRow('及物性', entry.transitivity),
                ],
              ),
            ),
          if (entry.family.isNotEmpty)
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('词族派生', icon: Icons.hub_outlined),
                  ...entry.family.map((f) => Padding(
                        padding: const EdgeInsets.symmetric(vertical: 5),
                        child: ArabicText(f, size: 17),
                      )),
                ],
              ),
            ),
          if (entry.phrases.isNotEmpty)
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('常用短语', icon: Icons.short_text),
                  ...entry.phrases.map((p) => Padding(
                        padding: const EdgeInsets.symmetric(vertical: 5),
                        child: ArabicText(p, size: 17),
                      )),
                ],
              ),
            ),
          if (entry.examples.isNotEmpty)
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('例句', icon: Icons.format_quote_outlined),
                  ...entry.examples.map((ex) => Padding(
                        padding: const EdgeInsets.only(bottom: 12),
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.stretch,
                          children: [
                            ArabicText(ex.ar, size: 19),
                            const SizedBox(height: 4),
                            Text(ex.zh,
                                style: const TextStyle(
                                    fontSize: 13, color: Color(0xFF68786F))),
                          ],
                        ),
                      )),
                ],
              ),
            ),
        ],
      ),
    );
  }
}

// =====================================================================
// 十、页面：句子解析
// =====================================================================
class TranslatePage extends StatefulWidget {
  const TranslatePage({super.key});
  @override
  State<TranslatePage> createState() => _TranslatePageState();
}

class _TranslatePageState extends State<TranslatePage> {
  final _ctrl = TextEditingController();
  SentenceAnalysis? _result;

  static const _samples = [
    'كَتَبَ الطَّالِبُ الدَّرْسَ.',
    'أَقْرَأُ الْكِتَابَ فِي الْمَكْتَبَةِ.',
    'ذَهَبَ أَحْمَدُ إِلَى السُّوقِ.',
    'إِنَّ اللَّغَةَ الْعَرَبِيَّةَ وَاسِعَةٌ.',
  ];

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  void _analyze() {
    final t = _ctrl.text.trim();
    if (t.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('请输入或粘贴阿拉伯语句子')),
      );
      return;
    }
    FocusScope.of(context).unfocus();
    setState(() => _result = analyzeSentence(t));
  }

  @override
  Widget build(BuildContext context) {
    final r = _result;
    return Scaffold(
      appBar: AppBar(title: const Text('句子解析')),
      body: ListView(
        padding: const EdgeInsets.fromLTRB(14, 12, 14, 30),
        children: [
          AppCard(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                TextField(
                  controller: _ctrl,
                  maxLines: 4,
                  minLines: 3,
                  textDirection: TextDirection.rtl,
                  style: const TextStyle(fontSize: 20, height: 1.9),
                  decoration: InputDecoration(
                    hintText: '在此粘贴阿拉伯语句子…',
                    border: OutlineInputBorder(
                        borderRadius: BorderRadius.circular(10)),
                    contentPadding: const EdgeInsets.all(12),
                  ),
                ),
                const SizedBox(height: 10),
                FilledButton.icon(
                  onPressed: _analyze,
                  icon: const Icon(Icons.auto_awesome),
                  label: const Text('解析句子'),
                ),
              ],
            ),
          ),
          if (r == null)
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('试试这些例句', icon: Icons.lightbulb_outline),
                  Wrap(
                    spacing: 8,
                    runSpacing: 8,
                    children: _samples
                        .map((s) => ActionChip(
                              label: Directionality(
                                textDirection: TextDirection.rtl,
                                child: Text(s,
                                    style: const TextStyle(fontSize: 13)),
                              ),
                              onPressed: () {
                                _ctrl.text = s;
                                _analyze();
                              },
                            ))
                        .toList(),
                  ),
                ],
              ),
            ),
          if (r != null) ...[
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('① 整句（带完整元音符）',
                      icon: Icons.text_fields),
                  ArabicText(r.voweled, size: 24, align: TextAlign.right),
                ],
              ),
            ),
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('② 中文翻译', icon: Icons.translate),
                  Text(r.translation,
                      style: const TextStyle(fontSize: 15, height: 1.75)),
                ],
              ),
            ),
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('③ 逐词拆解', icon: Icons.list_alt),
                  ...r.words.asMap().entries.map((e) {
                    final i = e.key;
                    final w = e.value;
                    return Container(
                      padding: const EdgeInsets.symmetric(vertical: 10),
                      decoration: BoxDecoration(
                        border: Border(
                          top: BorderSide(
                            color: i == 0
                                ? Colors.transparent
                                : const Color(0xFFEEF1F0),
                          ),
                        ),
                      ),
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Row(
                            children: [
                              Text('${i + 1}.',
                                  style: const TextStyle(
                                      fontSize: 13, color: Color(0xFF9AA8A4))),
                              const SizedBox(width: 8),
                              Expanded(child: ArabicText(w.surface, size: 22)),
                            ],
                          ),
                          const SizedBox(height: 6),
                          _tag('词性', w.pos),
                          _tag('格位', w.caseInfo),
                          _tag('句法作用', w.role),
                          if (w.note.isNotEmpty) _tag('释义', w.note),
                        ],
                      ),
                    );
                  }),
                ],
              ),
            ),
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('④ 句法分析',
                      icon: Icons.account_tree_outlined),
                  ...r.syntax.map((s) => Padding(
                        padding: const EdgeInsets.symmetric(vertical: 4),
                        child: Row(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            const Padding(
                              padding: EdgeInsets.only(top: 6, right: 8),
                              child: Icon(Icons.circle,
                                  size: 6, color: Color(0xFF1E6F5C)),
                            ),
                            Expanded(
                              child: Text(s,
                                  style: const TextStyle(
                                      fontSize: 14, height: 1.7)),
                            ),
                          ],
                        ),
                      )),
                ],
              ),
            ),
            AppCard(
              child: Row(
                children: [
                  const Icon(Icons.star_border, color: Color(0xFFF0A500)),
                  const SizedBox(width: 8),
                  const Expanded(child: Text('收藏本次解析结果')),
                  FavButton(FavItem(
                    type: '句子',
                    title: r.voweled,
                    content: r.translation,
                  )),
                ],
              ),
            ),
          ],
        ],
      ),
    );
  }

  Widget _tag(String label, String value) {
    return Padding(
      padding: const EdgeInsets.only(bottom: 4, left: 20),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Container(
            margin: const EdgeInsets.only(top: 2, right: 6),
            padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 1),
            decoration: BoxDecoration(
              color: const Color(0xFFE8F2EF),
              borderRadius: BorderRadius.circular(4),
            ),
            child: Text(label,
                style: const TextStyle(
                    fontSize: 11, color: Color(0xFF1E6F5C))),
          ),
          Expanded(
            child: Text(value,
                style: const TextStyle(fontSize: 13, height: 1.6)),
          ),
        ],
      ),
    );
  }
}

// =====================================================================
// 十一、页面：工具
// =====================================================================
class ToolsPage extends StatelessWidget {
  const ToolsPage({super.key});
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('词法工具')),
      body: ListView(
        padding: const EdgeInsets.fromLTRB(14, 12, 14, 30),
        children: [
          _tool(context,
              icon: Icons.table_chart_outlined,
              title: '三母动词变位表',
              subtitle: '过去式 · 现在式 · 命令式（全套变位）',
              page: const ConjugationPage()),
          _tool(context,
              icon: Icons.group_work_outlined,
              title: '名词复数查询',
              subtitle: '单数 → 破碎复数 / 完整复数 对照',
              page: const PluralPage()),
          _tool(context,
              icon: Icons.link_outlined,
              title: '常用虚词·连词用法',
              subtitle: 'إِلَّا أَنَّ / لَكِنَّ / كَانَ / لَمْ / لَنْ 等',
              page: const ParticlePage()),
        ],
      ),
    );
  }

  Widget _tool(BuildContext context,
      {required IconData icon,
      required String title,
      required String subtitle,
      required Widget page}) {
    return AppCard(
      child: InkWell(
        onTap: () => Navigator.push(
            context, MaterialPageRoute(builder: (_) => page)),
        child: Row(
          children: [
            Container(
              padding: const EdgeInsets.all(10),
              decoration: BoxDecoration(
                color: const Color(0xFFE8F2EF),
                borderRadius: BorderRadius.circular(10),
              ),
              child: Icon(icon, color: const Color(0xFF1E6F5C)),
            ),
            const SizedBox(width: 12),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(title,
                      style: const TextStyle(
                          fontSize: 15, fontWeight: FontWeight.w600)),
                  const SizedBox(height: 3),
                  Text(subtitle,
                      style: const TextStyle(
                          fontSize: 12, color: Color(0xFF8B9A96))),
                ],
              ),
            ),
            const Icon(Icons.chevron_right, color: Color(0xFFB9C4C1)),
          ],
        ),
      ),
    );
  }
}

const List<List<String>> kVerbPresets = [
  ['كَتَبَ', 'يَكْتُبُ', '写'],
  ['ذَهَبَ', 'يَذْهَبُ', '去'],
  ['قَرَأَ', 'يَقْرَأُ', '读'],
  ['فَتَحَ', 'يَفْتَحُ', '打开'],
  ['جَلَسَ', 'يَجْلِسُ', '坐'],
  ['ضَرَبَ', 'يَضْرِبُ', '打'],
  ['نَصَرَ', 'يَنْصُرُ', '援助'],
  ['دَرَسَ', 'يَدْرُسُ', '学习'],
];

class ConjugationPage extends StatefulWidget {
  const ConjugationPage({super.key});
  @override
  State<ConjugationPage> createState() => _ConjugationPageState();
}

class _ConjugationPageState extends State<ConjugationPage> {
  final _pastCtrl = TextEditingController(text: 'كَتَبَ');
  final _presCtrl = TextEditingController(text: 'يَكْتُبُ');
  late ConjugationTable _table;

  @override
  void initState() {
    super.initState();
    _rebuild();
  }

  @override
  void dispose() {
    _pastCtrl.dispose();
    _presCtrl.dispose();
    super.dispose();
  }

  void _rebuild() {
    _table = buildConjugation(_pastCtrl.text, _presCtrl.text);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('三母动词变位表')),
      body: ListView(
        padding: const EdgeInsets.fromLTRB(14, 12, 14, 30),
        children: [
          AppCard(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                const SectionTitle('选择或输入动词', icon: Icons.edit_outlined),
                const SizedBox(height: 6),
                Wrap(
                  spacing: 8,
                  runSpacing: 8,
                  children: kVerbPresets
                      .map((v) => ActionChip(
                            label: Text('${v[0]}（${v[2]}）'),
                            onPressed: () {
                              setState(() {
                                _pastCtrl.text = v[0];
                                _presCtrl.text = v[1];
                                _rebuild();
                              });
                            },
                          ))
                      .toList(),
                ),
                const SizedBox(height: 12),
                TextField(
                  controller: _pastCtrl,
                  textDirection: TextDirection.rtl,
                  style: const TextStyle(fontSize: 18, height: 1.8),
                  decoration: const InputDecoration(
                    labelText: '过去式（第三人称阳性单数）',
                    border: OutlineInputBorder(),
                    isDense: true,
                  ),
                  onChanged: (_) => setState(_rebuild),
                ),
                const SizedBox(height: 10),
                TextField(
                  controller: _presCtrl,
                  textDirection: TextDirection.rtl,
                  style: const TextStyle(fontSize: 18, height: 1.8),
                  decoration: const InputDecoration(
                    labelText: '现在式（第三人称阳性单数）',
                    border: OutlineInputBorder(),
                    isDense: true,
                  ),
                  onChanged: (_) => setState(_rebuild),
                ),
              ],
            ),
          ),
          _tableCard(_table.pastTitle, _table.pastRows),
          _tableCard(_table.presentTitle, _table.presentRows),
          _tableCard(_table.imperativeTitle, _table.imperativeRows),
          const AppCard(
            child: Text(
              '说明：本表按《新编阿拉伯语》教材体系，对规则三母健全动词自动生成。'
              '中空、重母、带 Hamza 动词的部分形式存在特殊规则，请以教材为准。',
              style: TextStyle(
                  fontSize: 12, color: Color(0xFF8B9A96), height: 1.7),
            ),
          ),
        ],
      ),
    );
  }

  Widget _tableCard(String title, List<ConjugationRow> rows) {
    return AppCard(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          SectionTitle(title, icon: Icons.grid_on_outlined),
          ...rows.asMap().entries.map((e) {
            final i = e.key;
            final r = e.value;
            return Container(
              padding: const EdgeInsets.symmetric(vertical: 9),
              decoration: BoxDecoration(
                border: Border(
                  top: BorderSide(
                    color: i == 0
                        ? Colors.transparent
                        : const Color(0xFFEEF1F0),
                  ),
                ),
              ),
              child: Row(
                children: [
                  SizedBox(width: 96, child: ArabicText(r.pronoun, size: 17)),
                  const SizedBox(width: 8),
                  Expanded(child: ArabicText(r.form, size: 22)),
                ],
              ),
            );
          }),
        ],
      ),
    );
  }
}

class PluralPage extends StatefulWidget {
  const PluralPage({super.key});
  @override
  State<PluralPage> createState() => _PluralPageState();
}

class _PluralPageState extends State<PluralPage> {
  final _ctrl = TextEditingController();
  late List<WordEntry> _list;

  @override
  void initState() {
    super.initState();
    _list = kDictionary.where((e) => e.isNoun).toList();
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  void _filter(String q) {
    final query = q.trim();
    final nouns = kDictionary.where((e) => e.isNoun);
    if (query.isEmpty) {
      setState(() => _list = nouns.toList());
    } else {
      setState(() {
        _list = nouns
            .where((e) => e.zh.contains(query) || e.singular.contains(query))
            .toList();
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('名词复数查询')),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.fromLTRB(14, 12, 14, 8),
            child: TextField(
              controller: _ctrl,
              onChanged: _filter,
              decoration: InputDecoration(
                hintText: '输入中文或单数名词',
                prefixIcon: const Icon(Icons.search),
                filled: true,
                fillColor: Colors.white,
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(12),
                  borderSide: BorderSide.none,
                ),
                contentPadding: const EdgeInsets.symmetric(vertical: 4),
              ),
            ),
          ),
          Expanded(
            child: ListView.builder(
              padding: const EdgeInsets.fromLTRB(14, 4, 14, 20),
              itemCount: _list.length,
              itemBuilder: (context, i) {
                final e = _list[i];
                return AppCard(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Row(
                        children: [
                          Expanded(child: ArabicText(e.singular, size: 22)),
                          Text(e.zh,
                              style: const TextStyle(
                                  fontSize: 13, color: Color(0xFF68786F))),
                        ],
                      ),
                      const Divider(height: 18),
                      if (e.plural.isNotEmpty) _line('破碎复数', e.plural),
                      if (e.plural2.isNotEmpty) _line('完整复数', e.plural2),
                      if (e.gender.isNotEmpty) _line('性', e.gender),
                    ],
                  ),
                );
              },
            ),
          ),
        ],
      ),
    );
  }

  Widget _line(String label, String value) => Padding(
        padding: const EdgeInsets.symmetric(vertical: 4),
        child: Row(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            SizedBox(
              width: 68,
              child: Text(label,
                  style: const TextStyle(
                      fontSize: 12, color: Color(0xFF8B9A96))),
            ),
            Expanded(child: ArabicText(value, size: 18)),
          ],
        ),
      );
}

class ParticlePage extends StatelessWidget {
  const ParticlePage({super.key});
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('常用虚词·连词')),
      body: ListView.builder(
        padding: const EdgeInsets.fromLTRB(14, 12, 14, 30),
        itemCount: kParticleItems.length,
        itemBuilder: (context, i) {
          final p = kParticleItems[i];
          return AppCard(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  children: [
                    Expanded(child: ArabicText(p.word, size: 24)),
                    Container(
                      padding: const EdgeInsets.symmetric(
                          horizontal: 8, vertical: 3),
                      decoration: BoxDecoration(
                        color: const Color(0xFFE8F2EF),
                        borderRadius: BorderRadius.circular(6),
                      ),
                      child: Text(p.type,
                          style: const TextStyle(
                              fontSize: 11, color: Color(0xFF1E6F5C))),
                    ),
                  ],
                ),
                const SizedBox(height: 8),
                Text(p.usage,
                    style: const TextStyle(fontSize: 14, height: 1.75)),
                if (p.examples.isNotEmpty) ...[
                  const Divider(height: 22),
                  ...p.examples.map((ex) => Padding(
                        padding: const EdgeInsets.only(bottom: 8),
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.stretch,
                          children: [
                            ArabicText(ex.ar, size: 18),
                            const SizedBox(height: 3),
                            Text(ex.zh,
                                style: const TextStyle(
                                    fontSize: 13, color: Color(0xFF68786F))),
                          ],
                        ),
                      )),
                ],
              ],
            ),
          );
        },
      ),
    );
  }
}

// =====================================================================
// 十二、页面：知识库
// =====================================================================
class KnowledgePage extends StatelessWidget {
  const KnowledgePage({super.key});
  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 2,
      child: Scaffold(
        appBar: AppBar(
          title: const Text('知识库'),
          bottom: const TabBar(tabs: [Tab(text: '语法知识'), Tab(text: '历史知识')]),
        ),
        body: const TabBarView(
          children: [
            _KnowledgeList(items: kGrammarItems),
            _KnowledgeList(items: kHistoryItems),
          ],
        ),
      ),
    );
  }
}

class _KnowledgeList extends StatefulWidget {
  final List<KnowledgeItem> items;
  const _KnowledgeList({required this.items});
  @override
  State<_KnowledgeList> createState() => _KnowledgeListState();
}

class _KnowledgeListState extends State<_KnowledgeList> {
  String _category = '全部';

  @override
  Widget build(BuildContext context) {
    final cats = <String>['全部'];
    for (final it in widget.items) {
      if (!cats.contains(it.category)) cats.add(it.category);
    }
    final list = _category == '全部'
        ? widget.items
        : widget.items.where((e) => e.category == _category).toList();

    return Column(
      children: [
        SizedBox(
          height: 46,
          child: ListView(
            scrollDirection: Axis.horizontal,
            padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
            children: cats
                .map((c) => Padding(
                      padding: const EdgeInsets.only(right: 8),
                      child: ChoiceChip(
                        label: Text(c),
                        selected: _category == c,
                        onSelected: (_) => setState(() => _category = c),
                      ),
                    ))
                .toList(),
          ),
        ),
        Expanded(
          child: ListView.builder(
            padding: const EdgeInsets.fromLTRB(14, 6, 14, 30),
            itemCount: list.length,
            itemBuilder: (context, i) {
              final it = list[i];
              return AppCard(
                child: InkWell(
                  onTap: () => Navigator.push(
                    context,
                    MaterialPageRoute(
                        builder: (_) => KnowledgeDetailPage(item: it)),
                  ),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Row(
                        children: [
                          Container(
                            padding: const EdgeInsets.symmetric(
                                horizontal: 8, vertical: 3),
                            decoration: BoxDecoration(
                              color: const Color(0xFFE8F2EF),
                              borderRadius: BorderRadius.circular(6),
                            ),
                            child: Text(it.category,
                                style: const TextStyle(
                                    fontSize: 11, color: Color(0xFF1E6F5C))),
                          ),
                          const Spacer(),
                          const Icon(Icons.chevron_right,
                              color: Color(0xFFB9C4C1)),
                        ],
                      ),
                      const SizedBox(height: 8),
                      Text(it.title,
                          style: const TextStyle(
                              fontSize: 15, fontWeight: FontWeight.w600)),
                      if (it.subtitle.isNotEmpty) ...[
                        const SizedBox(height: 3),
                        Text(it.subtitle,
                            style: const TextStyle(
                                fontSize: 12, color: Color(0xFF8B9A96))),
                      ],
                    ],
                  ),
                ),
              );
            },
          ),
        ),
      ],
    );
  }
}

class KnowledgeDetailPage extends StatelessWidget {
  final KnowledgeItem item;
  const KnowledgeDetailPage({super.key, required this.item});
  @override
  Widget build(BuildContext context) {
    final fav = FavItem(
      type: '语法',
      title: item.title,
      content: item.points.join('\n'),
    );
    return Scaffold(
      appBar: AppBar(
        title: Text(item.title, style: const TextStyle(fontSize: 16)),
        actions: [FavButton(fav)],
      ),
      body: ListView(
        padding: const EdgeInsets.fromLTRB(14, 12, 14, 30),
        children: [
          if (item.subtitle.isNotEmpty)
            Padding(
              padding: const EdgeInsets.only(bottom: 10, left: 4),
              child: Text(item.subtitle,
                  style: const TextStyle(
                      fontSize: 12, color: Color(0xFF8B9A96))),
            ),
          AppCard(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                const SectionTitle('要点', icon: Icons.list_alt),
                ...item.points.map((p) => Padding(
                      padding: const EdgeInsets.symmetric(vertical: 6),
                      child: Row(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Container(
                            margin: const EdgeInsets.only(top: 6, right: 8),
                            width: 6,
                            height: 6,
                            decoration: const BoxDecoration(
                              color: Color(0xFF1E6F5C),
                              shape: BoxShape.circle,
                            ),
                          ),
                          Expanded(
                            child: Text(p,
                                style: const TextStyle(
                                    fontSize: 14, height: 1.8)),
                          ),
                        ],
                      ),
                    )),
              ],
            ),
          ),
          if (item.examples.isNotEmpty)
            AppCard(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const SectionTitle('例句', icon: Icons.format_quote_outlined),
                  ...item.examples.map((ex) => Padding(
                        padding: const EdgeInsets.only(bottom: 14),
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.stretch,
                          children: [
                            ArabicText(ex.ar, size: 20),
                            const SizedBox(height: 4),
                            Text(ex.zh,
                                style: const TextStyle(
                                    fontSize: 13, color: Color(0xFF68786F))),
                          ],
                        ),
                      )),
                ],
              ),
            ),
        ],
      ),
    );
  }
}

// =====================================================================
// 十三、页面：我的
// =====================================================================
class ProfilePage extends StatelessWidget {
  const ProfilePage({super.key});
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('我的收藏'),
        actions: [
          ListenableBuilder(
            listenable: FavoritesStore.instance,
            builder: (context, _) {
              final empty = FavoritesStore.instance.items.isEmpty;
              return IconButton(
                tooltip: '清空收藏',
                icon: const Icon(Icons.delete_sweep_outlined),
                onPressed: empty
                    ? null
                    : () async {
                        final ok = await showDialog<bool>(
                          context: context,
                          builder: (ctx) => AlertDialog(
                            title: const Text('清空收藏'),
                            content: const Text('确定要清空全部收藏吗？此操作不可撤销。'),
                            actions: [
                              TextButton(
                                onPressed: () => Navigator.pop(ctx, false),
                                child: const Text('取消'),
                              ),
                              FilledButton(
                                onPressed: () => Navigator.pop(ctx, true),
                                child: const Text('清空'),
                              ),
                            ],
                          ),
                        );
                        if (ok == true) FavoritesStore.instance.clear();
                      },
              );
            },
          ),
        ],
      ),
      body: ListenableBuilder(
        listenable: FavoritesStore.instance,
        builder: (context, _) {
          final items = FavoritesStore.instance.items;
          if (items.isEmpty) {
            return const Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Icon(Icons.star_border, size: 56, color: Color(0xFFCBD5D2)),
                  SizedBox(height: 12),
                  Text('还没有收藏内容',
                      style: TextStyle(color: Color(0xFF9AA8A4))),
                  SizedBox(height: 6),
                  Text('在词典、解析、知识页点击 ☆ 即可收藏',
                      style: TextStyle(fontSize: 12, color: Color(0xFFB9C4C1))),
                ],
              ),
            );
          }
          return ListView.builder(
            padding: const EdgeInsets.fromLTRB(14, 12, 14, 30),
            itemCount: items.length,
            itemBuilder: (context, i) {
              final it = items[i];
              return Container(
                margin: const EdgeInsets.only(bottom: 12),
                padding: const EdgeInsets.all(14),
                decoration: BoxDecoration(
                  color: Colors.white,
                  borderRadius: BorderRadius.circular(14),
                  boxShadow: const [
                    BoxShadow(
                        color: Color(0x0A000000),
                        blurRadius: 8,
                        offset: Offset(0, 2)),
                  ],
                ),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Row(
                      children: [
                        Container(
                          padding: const EdgeInsets.symmetric(
                              horizontal: 8, vertical: 3),
                          decoration: BoxDecoration(
                            color: const Color(0xFFE8F2EF),
                            borderRadius: BorderRadius.circular(6),
                          ),
                          child: Text(it.type,
                              style: const TextStyle(
                                  fontSize: 11, color: Color(0xFF1E6F5C))),
                        ),
                        const Spacer(),
                        IconButton(
                          visualDensity: VisualDensity.compact,
                          icon: const Icon(Icons.delete_outline, size: 20),
                          onPressed: () => FavoritesStore.instance.removeAt(i),
                        ),
                      ],
                    ),
                    const SizedBox(height: 4),
                    Directionality(
                      textDirection: TextDirection.rtl,
                      child: Text(it.title,
                          textAlign: TextAlign.right,
                          style: const TextStyle(
                              fontSize: 18,
                              height: 1.9,
                              fontWeight: FontWeight.w600)),
                    ),
                    const SizedBox(height: 6),
                    Text(it.content,
                        style: const TextStyle(
                            fontSize: 13, height: 1.7, color: Color(0xFF68786F))),
                  ],
                ),
              );
            },
          );
        },
      ),
    );
  }
}

// =====================================================================
// 十四、主入口
// =====================================================================
void main() => runApp(const ArabicApp());

class ArabicApp extends StatelessWidget {
  const ArabicApp({super.key});
  @override
  Widget build(BuildContext context) {
    const seed = Color(0xFF1E6F5C);
    return MaterialApp(
      title: '阿拉伯语学习助手',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(seedColor: seed),
        scaffoldBackgroundColor: const Color(0xFFF6F7F9),
        appBarTheme: const AppBarTheme(
          centerTitle: true,
          elevation: 0,
          scrolledUnderElevation: 1,
          backgroundColor: Colors.white,
          foregroundColor: Color(0xFF1E3A34),
        ),
      ),
      home: const RootScaffold(),
    );
  }
}

class RootScaffold extends StatefulWidget {
  const RootScaffold({super.key});
  @override
  State<RootScaffold> createState() => _RootScaffoldState();
}

class _RootScaffoldState extends State<RootScaffold> {
  int _index = 0;
  static const _pages = <Widget>[
    DictionaryPage(),
    TranslatePage(),
    ToolsPage(),
    KnowledgePage(),
    ProfilePage(),
  ];
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: IndexedStack(index: _index, children: _pages),
      bottomNavigationBar: NavigationBar(
        selectedIndex: _index,
        height: 62,
        labelBehavior: NavigationDestinationLabelBehavior.alwaysShow,
        onDestinationSelected: (i) => setState(() => _index = i),
        destinations: const [
          NavigationDestination(
              icon: Icon(Icons.menu_book_outlined),
              selectedIcon: Icon(Icons.menu_book),
              label: '词典'),
          NavigationDestination(
              icon: Icon(Icons.translate_outlined),
              selectedIcon: Icon(Icons.translate),
              label: '解析'),
          NavigationDestination(
              icon: Icon(Icons.handyman_outlined),
              selectedIcon: Icon(Icons.handyman),
              label: '工具'),
          NavigationDestination(
              icon: Icon(Icons.school_outlined),
              selectedIcon: Icon(Icons.school),
              label: '知识'),
          NavigationDestination(
              icon: Icon(Icons.person_outline),
              selectedIcon: Icon(Icons.person),
              label: '我的'),
        ],
      ),
    );
  }
}
