/* StudyNova XP, Daily Streaks, Badges, and Quiz Party Mode */

(function () {
    "use strict";

    const PREFIX = "studyNova_";
    const MAX_LESSONS = 5;

    function readNumber(key) {
        const value = Number(localStorage.getItem(key));
        return Number.isFinite(value) && value >= 0 ? value : 0;
    }

    function readJSON(key) {
        try {
            const value = JSON.parse(localStorage.getItem(key) || "{}");

            return value && typeof value === "object" && !Array.isArray(value)
                ? value
                : {};
        } catch (error) {
            return {};
        }
    }

    function saveJSON(key, value) {
        localStorage.setItem(key, JSON.stringify(value));
    }

    function todayString() {
        const date = new Date();

        return date.getFullYear() + "-" +
            String(date.getMonth() + 1).padStart(2, "0") + "-" +
            String(date.getDate()).padStart(2, "0");
    }

    function daysBetween(first, second) {
        const a = new Date(first + "T12:00:00");
        const b = new Date(second + "T12:00:00");

        return Math.round((b - a) / 86400000);
    }

    /* XP is awarded only once for each lesson or quiz. */

    function awardXP(id, points) {
        const awards = readJSON(PREFIX + "xpAwards");

        if (Object.prototype.hasOwnProperty.call(awards, id)) {
            return false;
        }

        awards[id] = points;
        saveJSON(PREFIX + "xpAwards", awards);

        const newXP = readNumber(PREFIX + "xp") + points;

        localStorage.setItem(PREFIX + "xp", String(newXP));

        return true;
    }

    /* Keep the streak to one update per calendar day. */

    function updateDailyStreak() {
        const today = todayString();
        const lastDay = localStorage.getItem(PREFIX + "lastActiveDay");

        let streak = readNumber(PREFIX + "streak");

        if (lastDay === today) {
            return;
        }

        if (!lastDay) {
            streak = 1;
        } else {
            const difference = daysBetween(lastDay, today);

            if (difference === 1) {
                streak += 1;
            } else {
                streak = 1;
            }
        }

        localStorage.setItem(PREFIX + "streak", String(streak));
        localStorage.setItem(PREFIX + "lastActiveDay", today);
    }

    function processCompletedLessons() {
        for (let i = 1; i <= MAX_LESSONS; i++) {
            if (localStorage.getItem("completedLesson" + i) === "true") {
                awardXP("webLesson" + i, 20);
            }

            if (localStorage.getItem("completedAILesson" + i) === "true") {
                awardXP("aiLesson" + i, 20);
            }
        }
    }

    function processQuizScores() {
        for (let i = 1; i <= MAX_LESSONS; i++) {
            processOneQuiz("quiz" + i, 15);

            processOneQuiz(
                "aiQuiz" + i,
                i === 1 ? 10 : 15
            );
        }
    }

    function processOneQuiz(quizKey, defaultTotal) {
        const rawScore = localStorage.getItem(quizKey + "Score");

        if (rawScore === null) {
            return;
        }

        const score = Number(rawScore);
        const savedTotal = Number(
            localStorage.getItem(quizKey + "Total")
        );

        const total = savedTotal > 0 ? savedTotal : defaultTotal;

        if (
            !Number.isFinite(score) ||
            !Number.isFinite(total) ||
            total <= 0 ||
            score < 0 ||
            score > total
        ) {
            return;
        }

        const percentage = score / total;
        const points = percentage >= 0.8 ? 15 : 5;

        awardXP(quizKey, points);
    }

    function countCompletedLessons(prefix) {
        let count = 0;

        for (let i = 1; i <= MAX_LESSONS; i++) {
            if (localStorage.getItem(prefix + i) === "true") {
                count++;
            }
        }

        return count;
    }

    function hasHighWebQuizScore() {
        for (let i = 1; i <= MAX_LESSONS; i++) {
            const rawScore = localStorage.getItem("quiz" + i + "Score");

            if (rawScore === null) {
                continue;
            }

            const score = Number(rawScore);
            const savedTotal = Number(
                localStorage.getItem("quiz" + i + "Total")
            );
            const total = savedTotal > 0 ? savedTotal : 15;

            if (
                Number.isFinite(score) &&
                Number.isFinite(total) &&
                total > 0 &&
                score >= 0 &&
                score <= total &&
                score / total >= 0.8
            ) {
                return true;
            }
        }

        return false;
    }

    /* Update shared badges and keep compatibility with the existing page. */

    function updateBadges() {
        const badges = readJSON(PREFIX + "badges");

        const webLessons = countCompletedLessons("completedLesson");
        const aiLessons = countCompletedLessons("completedAILesson");
        const quizAwards = readJSON(PREFIX + "xpAwards");

        const quizCount = Object.keys(quizAwards).filter(function (key) {
            return /^(quiz|aiQuiz)\d+$/.test(key);
        }).length;

        if (webLessons + aiLessons >= 1) {
            badges.firstLesson = true;
        }

        if (quizCount >= 1) {
            badges.quizStarter = true;
        }

        if (readNumber(PREFIX + "streak") >= 3) {
            badges.streak3 = true;
        }

        if (webLessons === MAX_LESSONS) {
            badges.webGraduate = true;
        }

        if (aiLessons === MAX_LESSONS) {
            badges.aiGraduate = true;
        }

        if (readNumber(PREFIX + "streak") >= 7) {
            badges.learningChampion = true;
        }

        if (hasHighWebQuizScore()) {
            badges.quizMaster = true;
        }

        saveJSON(PREFIX + "badges", badges);

        // These keys are also used by the existing achievements page.
        if (badges.webGraduate) {
            localStorage.setItem("webDevelopmentBadge", "true");
        }

        if (badges.aiGraduate) {
            localStorage.setItem("aiGraduateBadge", "true");
        }

        if (badges.learningChampion) {
            localStorage.setItem("learningChampionBadge", "true");
        }

        if (badges.quizMaster) {
            localStorage.setItem("quizMasterBadge", "true");
        }
    }

    function refreshDashboard() {
        const dashboard = document.getElementById(
            "studyNovaMiniDashboard"
        );

        if (!dashboard) {
            return;
        }

        dashboard.textContent =
            "⭐ StudyNova XP: " + readNumber(PREFIX + "xp") +
            "  |  🔥 Streak: " + readNumber(PREFIX + "streak") +
            " day(s)";
    }

    /* One shared celebration for high-scoring quizzes. */

    function celebrateQuiz(score, total) {
        score = Number(score);
        total = Number(total);

        if (
            !Number.isFinite(score) ||
            !Number.isFinite(total) ||
            total <= 0 ||
            score < 0 ||
            score > total ||
            score / total < 0.8
        ) {
            return;
        }

        // Do not create a second shared celebration while one is visible.
        if (document.getElementById("studyNovaPartyLayer")) {
            return;
        }

        const perfect = score === total;

        let style = document.getElementById("studyNovaPartyStyle");

        if (!style) {
            style = document.createElement("style");
            style.id = "studyNovaPartyStyle";

            style.textContent = `
                #studyNovaPartyLayer {
                    position: fixed;
                    inset: 0;
                    overflow: hidden;
                    pointer-events: none;
                    z-index: 999999;
                }

                .studyNovaConfetti {
                    position: absolute;
                    top: -24px;
                    width: 10px;
                    height: 16px;
                    border-radius: 2px;
                    animation: studyNovaFall linear forwards;
                }

                @keyframes studyNovaFall {
                    from { transform: translateY(-30px) rotate(0deg); }
                    to { transform: translateY(110vh) rotate(900deg); }
                }

                #studyNovaPartyMessage {
                    position: fixed;
                    top: 18%;
                    left: 50%;
                    transform: translateX(-50%);
                    width: min(90%, 480px);
                    padding: 24px;
                    border-radius: 20px;
                    background: white;
                    color: #3730a3;
                    text-align: center;
                    font-family: Arial, sans-serif;
                    font-size: 24px;
                    font-weight: bold;
                    box-shadow: 0 10px 40px #0003;
                    z-index: 1000000;
                    animation: studyNovaPop 0.5s ease-out;
                }

                #studyNovaPartyMessage small {
                    display: block;
                    margin-top: 10px;
                    font-size: 16px;
                    font-weight: normal;
                }

                @keyframes studyNovaPop {
                    from { opacity: 0; scale: 0.5; }
                    to { opacity: 1; scale: 1; }
                }

                @media (prefers-reduced-motion: reduce) {
                    .studyNovaConfetti {
                        animation-duration: 1s !important;
                    }
                }
            `;

            document.head.appendChild(style);
        }

        const layer = document.createElement("div");
        layer.id = "studyNovaPartyLayer";
        layer.setAttribute("aria-hidden", "true");

        const colors = [
            "#ef4444",
            "#f59e0b",
            "#22c55e",
            "#3b82f6",
            "#a855f7",
            "#ec4899"
        ];

        for (let i = 0; i < 110; i++) {
            const piece = document.createElement("span");

            piece.className = "studyNovaConfetti";
            piece.style.left = Math.random() * 100 + "%";
            piece.style.backgroundColor =
                colors[Math.floor(Math.random() * colors.length)];
            piece.style.animationDuration =
                (2 + Math.random() * 3) + "s";
            piece.style.animationDelay =
                (Math.random() * 1.2) + "s";

            layer.appendChild(piece);
        }

        document.body.appendChild(layer);

        const message = document.createElement("div");
        message.id = "studyNovaPartyMessage";
        message.setAttribute("role", "status");

        if (perfect) {
            message.innerHTML =
                "🏆 PERFECT SCORE! 🎉<small>" +
                score + " / " + total +
                " — Amazing work! You're a StudyNova star!</small>";
        } else {
            message.innerHTML =
                "🎉 CONGRATULATIONS! 🎊<small>" +
                score + " / " + total +
                " — Great job! You scored " +
                Math.round((score / total) * 100) + "%!</small>";
        }

        document.body.appendChild(message);

        window.setTimeout(function () {
            layer.remove();
            message.remove();
        }, 6000);
    }

    function refresh() {
        updateDailyStreak();
        processCompletedLessons();
        processQuizScores();
        updateBadges();
        refreshDashboard();
    }

    window.StudyNova = {
        refresh: refresh,
        celebrateQuiz: celebrateQuiz
    };

    if (document.readyState === "loading") {
        document.addEventListener("DOMContentLoaded", refresh);
    } else {
        refresh();
    }
})();