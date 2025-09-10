https://www.jitbit.com/alexblog/249-now-thats-what-i-call-a-hacker/ -----
import React, { useState, useMemo } from "react";
import { motion, AnimatePresence } from "framer-motion";
import {
  SKIN_TYPES,
  CONCERNS,
  SUB_CONCERNS,
  LIFESTYLE,
  ROUTINE_LEVELS,
  GOALS,
} from "../../utils/SkinInsightData";
import RecommendedProductCard from "./RecommendedProductCard";
import Fadeloader from "@/components/common/Fadeloader";
import api from "@/utils/api";

const pageVariants = {
  initial: { opacity: 0, y: 12 },
  animate: { opacity: 1, y: 0 },
  exit: { opacity: 0, y: -12 },
  transition: { duration: 0.25 },
};

const OptionCard = ({ active, label, onClick }) => (
  <motion.button
    type="button"
    whileHover={{ scale: 1.02 }}
    whileTap={{ scale: 0.97 }}
    onClick={onClick}
    className={`w-full rounded-xl border p-4 text-left transition
      ${active ? "border-[#8C6C54] bg-[#8C6C54]/10" : "border-zinc-200 bg-white hover:border-zinc-300"}`}
  >
    {label}
  </motion.button>
);

const MultiPill = ({ checked, label, onChange }) => (
  <motion.label
    whileHover={{ scale: 1.05 }}
    className={`cursor-pointer inline-flex items-center gap-2 rounded-full border px-4 py-2 text-sm
      ${checked ? "bg-[#8C6C54]/10 border-[#8C6C54] text-[#8C6C54]" : "bg-white border-zinc-200 text-zinc-700"}`}
  >
    <input type="checkbox" checked={checked} onChange={onChange} className="hidden" />
    {label}
  </motion.label>
);

export default function SkinInsightQuiz() {
  const [step, setStep] = useState(0);
  const [answers, setAnswers] = useState({
    ageGroup: "",
    email: "",
    skinType: "",
    concerns: [],
    subConcerns: [],
    lifestyle: [],
    routineLevel: "",
    goal: [],
  });
  const [errors, setErrors] = useState({});
  const [showResults, setShowResults] = useState(false);
  const [loading, setLoading] = useState(false);
  const [recommendations, setRecommendations] = useState([]);
  const [selectedProducts, setSelectedProducts] = useState([]);

  const steps = [
    {
      key: "ageEmail",
      title: "Tell us about you",
      render: () => (
        <>
          <div>
            <h3>How old are you?</h3>
            {["18-29", "30-39", "40-49", "50+"].map((opt) => (
              <OptionCard
                key={opt}
                active={answers.ageGroup === opt}
                label={opt}
                onClick={() => setAnswers((a) => ({ ...a, ageGroup: opt }))}
              />
            ))}
          </div>
          <div>
            <h3>Your Email</h3>
            <input
              value={answers.email}
              onChange={(e) => setAnswers((a) => ({ ...a, email: e.target.value }))}
              className="w-full border px-4 py-2 rounded-lg"
              placeholder="Enter your email"
            />
          </div>
        </>
      ),
      validate: () => {
        const errs = {};
        if (!answers.ageGroup) errs.ageGroup = "Pick your age";
        if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(answers.email)) errs.email = "Enter valid email";
        return errs;
      },
    },
    {
      key: "skinType",
      title: "What best describes your skin type?",
      options: SKIN_TYPES,
      type: "single",
    },
    {
      key: "concerns",
      title: "What are your main skincare concerns?",
      options: CONCERNS,
      type: "multi",
    },
    {
      key: "subConcerns",
      title: "Let’s go deeper — specific issues?",
      render: () =>
        answers.concerns.map((cid) => (
          <div key={cid}>
            <h4>{CONCERNS.find((c) => c.id === cid)?.label}</h4>
            {SUB_CONCERNS[cid]?.map((sub) => (
              <MultiPill
                key={sub}
                checked={answers.subConcerns.includes(sub)}
                label={sub}
                onChange={() =>
                  setAnswers((a) => ({
                    ...a,
                    subConcerns: a.subConcerns.includes(sub)
                      ? a.subConcerns.filter((x) => x !== sub)
                      : [...a.subConcerns, sub],
                  }))
                }
              />
            ))}
          </div>
        )),
    },
    { key: "lifestyle", title: "Which apply to your daily life?", options: LIFESTYLE, type: "multi" },
    { key: "routineLevel", title: "How’s your skincare routine?", options: ROUTINE_LEVELS, type: "single" },
    { key: "goal", title: "Your top skincare goal?", options: GOALS, type: "multi" },
  ];

  const handleNext = async () => {
    const current = steps[step];
    const errs = current.validate ? current.validate() : {};
    setErrors(errs);
    if (Object.keys(errs).length) return;
    if (step < steps.length - 1) setStep((s) => s + 1);
    else {
      setShowResults(true);
      await fetchRecommendations();
    }
  };

  const fetchRecommendations = async () => {
    try {
      setLoading(true);
      const res = await api.post("/recommendations/skin-form", {
        ...answers,
        skinTypes: [answers.skinType],
        routine: [answers.routineLevel],
      });
      const recs = res.data?.recommendations || [];
      setRecommendations(recs);
      setSelectedProducts(recs.map((p) => p.id || p._id));
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-3xl mx-auto p-6">
      {!showResults ? (
        <AnimatePresence mode="wait">
          <motion.div key={step} {...pageVariants} className="space-y-4">
            <h2 className="text-xl font-semibold">{steps[step].title}</h2>
            {steps[step].render ? (
              steps[step].render()
            ) : steps[step].type === "single" ? (
              steps[step].options.map((opt) => (
                <OptionCard
                  key={opt.id || opt}
                  active={answers[steps[step].key] === (opt.id || opt)}
                  label={opt.label || opt}
                  onClick={() => setAnswers((a) => ({ ...a, [steps[step].key]: opt.id || opt }))}
                />
              ))
            ) : (
              steps[step].options.map((opt) => {
                const checked = answers[steps[step].key].includes(opt.id);
                return (
                  <MultiPill
                    key={opt.id}
                    checked={checked}
                    label={opt.label}
                    onChange={() =>
                      setAnswers((a) => ({
                        ...a,
                        [steps[step].key]: checked
                          ? a[steps[step].key].filter((x) => x !== opt.id)
                          : [...a[steps[step].key], opt.id],
                      }))
                    }
                  />
                );
              })
            )}
            {errors[steps[step].key] && <p className="text-red-500">{errors[steps[step].key]}</p>}
            <div className="flex justify-between mt-6">
              <button disabled={step === 0} onClick={() => setStep((s) => s - 1)}>Back</button>
              <button onClick={handleNext}>{step < steps.length - 1 ? "Next" : "See Recommendations"}</button>
            </div>
          </motion.div>
        </AnimatePresence>
      ) : (
        <div>
          <h2>Your Personalized Products</h2>
          {loading ? (
            <Fadeloader />
          ) : (
            <div className="grid gap-4 md:grid-cols-3">
              {recommendations.map((p) => (
                <RecommendedProductCard key={p.id || p._id} product={p} selected={selectedProducts.includes(p.id)} />
              ))}
            </div>
          )}
        </div>
      )}
    </div>
  );
}
