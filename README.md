import { useState, useEffect } from "react";

const TypingEffect = () => {
  const [text, setText] = useState("");
  const fullText = "여기에 타이핑 될 문장을 넣으세요!";
  const [start, setStart] = useState(false);

  useEffect(() => {
    if (!start) return;

    let i = 0;
    const interval = setInterval(() => {
      setText((prev) => prev + fullText[i]);
      i++;
      if (i >= fullText.length) {
        clearInterval(interval);
      }
    }, 100); // 글자 나오는 속도(ms)

    return () => clearInterval(interval);
  }, [start]);

  useEffect(() => {
    // body 클릭 이벤트 등록
    const handleClick = () => {
      setText(""); // 다시 시작할 때 초기화 (원하면 제거 가능)
      setStart(true);
    };
    document.body.addEventListener("click", handleClick);

    return () => {
      document.body.removeEventListener("click", handleClick);
    };
  }, []);

  return (
    <div className="flex items-center justify-center h-screen bg-gray-900">
      <p className="text-white text-2xl font-mono whitespace-pre">{text}</p>
    </div>
  );
};

export default TypingEffect;