import { useState } from "react";
import { png2svg } from "svg-png-converter";

export default function PngToSvgConverter() {
  const [svg, setSvg] = useState("");

  const handleFile = async (e) => {
    const file = e.target.files[0];
    if (!file) return;

    const arrayBuffer = await file.arrayBuffer();
    const svgData = await png2svg({
      input: new Uint8Array(arrayBuffer),
      trace: true, // 벡터화 활성화
    });

    setSvg(svgData);
  };

  return (
    <div>
      <input type="file" accept="image/png" onChange={handleFile} />
      {svg && (
        <div
          style={{ border: "1px solid #ccc", marginTop: "10px" }}
          dangerouslySetInnerHTML={{ __html: svg }}
        />
      )}
    </div>
  );
}