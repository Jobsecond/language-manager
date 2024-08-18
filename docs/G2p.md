# G2p

## 简介

G2p可将自然语言转换为相应的音节，由语言分析器按语种自动调用。

## 创建一个G2p

### 1、创建G2p

基类： [IG2pFactory.h](../src/G2pMgr/IG2pFactory.h)

新建G2p类需公开继承基类IG2pFactory，类名建议为首字母大写的语言名称

G2p返回结构体

```c++
struct LangNote {
    QString lyric;
    QString syllable = QString();
    QStringList candidates = QStringList();
    QString language = "Unknown";
    QString category = "Unknown";
    bool g2pError = false;
    };
```

公共方法

```c++
Q_OBJECT    // 用于本地化的tr()函数
Phonic convert(const Note *&input, const QJsonObject *config) const;

// 耗时初始化操作在此进行
virtual bool initialize(QString &errMsg);

// 有个性化选项的G2p才需要重写，写法参照第3节
QJsonObject config() override;
QWidget *configWidget(QJsonObject *config) override;
```

样例

```c++
class Mandarin final : public IG2pFactory {
    Q_OBJECT    // 必须声明
public:
    Mandarin::Mandarin(QObject *parent) : IG2pFactory("Mandarin", parent) {
        setAuthor(tr("Xiao Lang"));         // 设置作者、带tr()用于本地化 
        setDisplayName(tr("Mandarin"));     // 设置本地化显示的G2p名称
        // 简介
        setDescription(tr("Using Pinyin as the phonetic notation method."));
    }
    ~Mandarin() override;

    bool initialize(QString &errMsg) override;  // 耗时初始化操作

    // 输入为字符串列表、当前G2p类的config
    QList<Phonic> convert(const QStringList &input, const QJsonObject *config) const override;

    // 自定义选项才需要重写
    QJsonObject config() override;
    QWidget *configWidget(QJsonObject *config) override;
};
```

### 2、添加G2p至管理器

[IG2pManager.h](../src/G2pMgr/IG2pManager.h)

```c++
IG2pManager::IG2pManager(IG2pManagerPrivate &d, QObject *parent) : QObject(parent), d_ptr(&d) {
    // ...
    
    // 添加新建G2p
    addG2p(new Xxx());
}
```

### 3、自定义选项

若G2p需要个性化选项，需重写以下方法

```c++
virtual QJsonObject config();
virtual QWidget *configWidget(QJsonObject *config);
```

样例

```c++
QJsonObject Mandarin::config() {
    QJsonObject config;
    config["tone"] = tone;
    config["convertNum"] = convertNum;
    return config;
}
// 返回QWidget *，面板包括个性化选项
QWidget *Mandarin::configWidget(QJsonObject *config) {
    // ...
    auto *toneCheckBox = new QCheckBox(tr("Tone"), widget);

    // 校验key
    if (config && config->keys().contains("tone")) {
        // 使用提供的变量值
        toneCheckBox->setChecked(config->value("tone").toBool());
    } else {
        // 使用默认值
        toneCheckBox->setChecked(tone);
    }

    // 选项变量需有默认值、连接控件信号修改变量值；发射Q_EMIT g2pConfigChanged()信号，用于更新持久化设置。
    connect(toneCheckBox, &QCheckBox::toggled, [this, config](const bool checked) {
        config->insert("tone", checked);
        Q_EMIT g2pConfigChanged();
    });
    return widget;
}
```