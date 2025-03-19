# EDITING-SOFTWERE
Adobe Premere Pro Demo App Code
#include <QApplication>
#include <QMainWindow>
#include <QPushButton>
#include <QVBoxLayout>
#include <QFileDialog>
#include <QMediaPlayer>
#include <QVideoWidget>
#include <QSlider>
#include <QHBoxLayout>
#include <QLabel>
#include <opencv2/opencv.hpp>
#include <QThread>

class SceneDetector : public QThread {
    Q_OBJECT
public:
    SceneDetector(const QString &videoPath, QObject *parent = nullptr) : QThread(parent), videoPath(videoPath) {}

signals:
    void sceneDetected(int timestamp);

protected:
    void run() override {
        cv::VideoCapture cap(videoPath.toStdString());
        if (!cap.isOpened()) return;

        cv::Mat prevFrame, frame, diff;
        int frameRate = cap.get(cv::CAP_PROP_FPS);
        int frameNumber = 0;

        while (cap.read(frame)) {
            if (!prevFrame.empty()) {
                cv::absdiff(frame, prevFrame, diff);
                double change = cv::sum(diff)[0] / (diff.rows * diff.cols);
                if (change > 30.0) {  // Threshold for scene change
                    emit sceneDetected(frameNumber / frameRate);
                }
            }
            frameNumber++;
            prevFrame = frame.clone();
        }
    }

private:
    QString videoPath;
};

class VideoEditor : public QMainWindow {
    Q_OBJECT

public:
    VideoEditor(QWidget *parent = nullptr) : QMainWindow(parent) {
        QWidget *centralWidget = new QWidget(this);
        QVBoxLayout *layout = new QVBoxLayout(centralWidget);

        QPushButton *importButton = new QPushButton("Import Video", this);
        layout->addWidget(importButton);

        videoWidget = new QVideoWidget(this);
        layout->addWidget(videoWidget);

        player = new QMediaPlayer(this);
        player->setVideoOutput(videoWidget);

        QHBoxLayout *controlsLayout = new QHBoxLayout();
        QPushButton *playButton = new QPushButton("Play", this);
        QPushButton *pauseButton = new QPushButton("Pause", this);
        controlsLayout->addWidget(playButton);
        controlsLayout->addWidget(pauseButton);
        layout->addLayout(controlsLayout);

        timelineSlider = new QSlider(Qt::Horizontal, this);
        layout->addWidget(timelineSlider);

        sceneLabel = new QLabel("Scenes: ", this);
        layout->addWidget(sceneLabel);

        connect(importButton, &QPushButton::clicked, this, &VideoEditor::importVideo);
        connect(playButton, &QPushButton::clicked, player, &QMediaPlayer::play);
        connect(pauseButton, &QPushButton::clicked, player, &QMediaPlayer::pause);
        connect(player, &QMediaPlayer::durationChanged, this, &VideoEditor::updateSliderRange);
        connect(player, &QMediaPlayer::positionChanged, this, &VideoEditor::updateSlider);
        connect(timelineSlider, &QSlider::sliderMoved, this, &VideoEditor::setPosition);

        setCentralWidget(centralWidget);
    }

private slots:
    void importVideo() {
        QString fileName = QFileDialog::getOpenFileName(this, "Open Video File", "", "Videos (*.mp4 *.avi *.mov)");
        if (!fileName.isEmpty()) {
            player->setMedia(QUrl::fromLocalFile(fileName));
            player->play();
            
            sceneDetector = new SceneDetector(fileName, this);
            connect(sceneDetector, &SceneDetector::sceneDetected, this, &VideoEditor::displayScene);
            sceneDetector->start();
        }
    }

    void updateSliderRange(qint64 duration) {
        timelineSlider->setRange(0, duration);
    }

    void updateSlider(qint64 position) {
        timelineSlider->setValue(position);
    }

    void setPosition(int position) {
        player->setPosition(position);
    }

    void displayScene(int timestamp) {
        sceneLabel->setText(sceneLabel->text() + QString::number(timestamp) + "s, ");
    }

private:
    QMediaPlayer *player;
    QVideoWidget *videoWidget;
    QSlider *timelineSlider;
    QLabel *sceneLabel;
    SceneDetector *sceneDetector;
};

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);
    VideoEditor editor;
    editor.show();
    return app.exec();
}

